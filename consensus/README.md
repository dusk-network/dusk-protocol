# Succinct Attestation
**Succinct Attestation** (**SA**) is a permissionless, committee-based[^1] Proof-of-Stake consensus protocol that provides statistical finality guarantees[^2]. 

The protocol is run by Dusk stakers, known as ***provisioners***, which are responsible for generating, validating, and finalizing new blocks.

Provisioners participate in turns to the production and validation of each new block of the ledger. Participation in each round is decided with a [*Deterministic Sortition*][ds] algorithm, which is used to extract a unique *block generator* and unique *voting committees* among provisioners, in a decentralized, non-interactive way.

## Notation
In this documentation we will use the following notation:

- $Latex$ is used for:
  - variables, with lowercase letters (e.g. a variable $v$)
  - simple objects, with uppercase letters (e.g. a committee $C$)
  - consensus parameters, state variables, and object fields, with whole words, (e.g. the $Quorum$ parameter)
- $\boldsymbol{Bold}$ is used for vectors (e.g. a bitset $\boldsymbol{bs}$)
- $\mathcal{Calligrafic}$ is used for actors (e.g. a provisioner $\mathcal{P}$)
- $\mathsf{Sans Serif}$ is used for structures (e.g. a block $\mathsf{B}$, or a $\mathsf{Reduction}$ message)
- *Italic* is used for functions names (e.g. the *Hash* function)

Moreover, we will conventionally use the following symbols:
- $pk$ and $sk$ are used to denote public and secret keys, respectively
- $\sigma$ characters are used for signatures (e.g. a message signature $\sigma_{M}$)
- $\eta$ characters are used for hash digests (e.g. a block hash $\eta_B$)
- $\tau$ characters are used for time variables (e.g. the Attestation timeout $\tau_{Attestation}$)
- $\mathcal{G}$ is used to the block generator
- $\mathsf{B}^c$ is used for the candidate block
- $\mathsf{B}^w$ is used for the winning block

While we can use the dot notation to specify a structure field (e.g. $\mathsf{B}.Header$), we will prefer the use of subscript when convenient (e.g. $\mathsf{H_B} = \mathsf{B}.Header$). Generally speaking, we use subscripts to specify the object or actor to which a variable belongs to (e.g. a block's hash $\eta_\mathsf{B}$), to indicate numeric order (e.g. the $i\text{th}$ block $`\mathsf{B}_i`$), or to specify a particular instantiation (e.g. *Hash*$`_{BLS}`$). In the case of vectors, we also use the standard notation $[i]$ to indicate the $i\text{th}$ element.

When using the dot notation, we will omit intermediate fields when possible. For instance, we will use $\mathsf{B}.Seed$ to denote $\mathsf{B}.Header.Seed$


We use $\leftarrow$ to assign object fields to separate variables. For instance, $\mathsf{H_B}, \boldsymbol{txs} \leftarrow \mathsf{B}$ will assign $\mathsf{B}.Header$ to $\mathsf{H_B}$, and $B.Transactions$ to $\boldsymbol{txs}$. The assignation follows the field order in the related structure definition. If a field is not needed in the algorithm, we ignore it by assigning it to a null variable $`\_`$.


## Protocol Overview
The SA protocol is executed in ***rounds***, with each round adding a new block to the chain.

Each round is composed of three phases:
  1. ***Attestation***: in this phase, a *generator*, extracted via [*DS*][ds], creates a new candidate block and broadcasts it to the network using a $\mathsf{NewBlock}$ message;
  
  2. ***Reduction***: in this phase, two separate committees of provisioners, extracted via [*DS*][ds], vote on the validity of the candidate block and broadcast their vote with a $\mathsf{Reduction}$ message. If positive votes reach a quorum of $\frac{2}{3}$ (67% of the votes) in each committee, an $\mathsf{Agreement}$ message is broadcast, which contains the aggregated votes of both committees. Conversely, the reduction fails and the protocol starts over (from *Attestation*) with a new generator and new voting committees (that is, a new candidate block is produced and voted upon);

  3. ***Ratification***: in this phase, all generated $\mathsf{Agreement}$ messages are collected; if messages for a specific candidate block reach a quorum (according to the second-reduction committee vote allocation), the candidate block is added to the chain along with a *certificate* containing the quorum votes from the $\mathsf{Agreement}$ message. This effectively ends the current round.

As both the *Attestation* and *Reduction* phases can fail, the *Attestation*/*Reduction* sequence can be executed multiple times within a single round (using different provisioners). We call each repetition of such sequence an ***iteration***, and each phase in the repetition a ***step***.
Note that since the *Reduction* phase has two consecutive votes, it is composed of two steps. Therefore, a single iteration is composed of three steps: (1) Attestation, (2) first Reduction, and (3) second Reduction.

A maximum number of 71 iterations (213 steps) is allowed to agree on a new valid block to add to the chain. If this number is exceeded, the network is considered unsafe and the consensus process is halted. Both step and iteration count starts from 1. 

### Candidate Block
A candidate block is the block generated in the [Attestation][att] step by the provisioner extracted as block generator. This is the block on which other provisioners will have to reach an agreement. If an agreement is not reached by the end of the iteration, a new candidate block will be produced and a new iteration will start.

Therefore, for each iteration, only one (valid) candidate block can be produced[^3]. To reflect this, we denote a candidate block with $\mathsf{B}^c_{r,i}$, where $r$ is the consensus round, and $i$ is the consensus iteration. 
Note that we simplify this notation to simply $\mathsf{B}^c$ when this does not generate confusion.

A candidate block that reaches an agreement is called a *winning* block.

## Provisioners and Stakes
<!-- TODO: link to Stake Contract -->
A provisioner is a user that locked a certain amount of their Dusk coins as *stake* (see [*Stake Contract*]()).
Formally, we define a *provisioner* $\mathcal{P}$ as:

$$\mathcal{P}=(pk_\mathcal{P},\boldsymbol{Stakes}_\mathcal{P}),$$

where $pk_\mathcal{P}$ is the BLS public key of the provisioner, and $\boldsymbol{Stakes}_\mathcal{P} = [S_0,\dots,S_n]$, where $S_i$ is a stake amount belonging to $\mathcal{P}$.

In turn, a *stake* is defined as:
$$S^\mathcal{P}=(Amount, Height),$$
where $\mathcal{P}$ is the provisioner that owns the stake, $Amount$ is the quantity of locked Dusks, and $Height$ is the height of the block where the lock action took place (i.e., when the *stake* transaction was included). The minimum value for a stake is defined by the [global parameter][cp] $MinStake$, and is currently equivalent to 1000 Dusk.

We say a stake $S$ is *mature* if it was locked more than two *epochs* before the current chain tip. Formally, given a stake $S$, we say $S$ is mature if 

$$Height_{Tip} \gt Height_S + (2 \times Epoch),$$

where $Height_{Tip}$ is the current tip's height, $Height_S$ is the stake's height, and $Epoch$ is a [global parameter][cp]. 

Moreover, we say a provisioner is *eligible* if it has at least one mature stake.

Note that only eligible provisioners participate in the consensus protocol.

## Consensus Messages
To run the SA protocol, participating nodes exchange consensus messages. There are four types of message:
- $\mathsf{NewBlock}$: this message stores a candidate block for a certain round and iteration. It is used during the [Attestation](./attestation/) phase;
- $\mathsf{Reduction}$: this message contains the vote of a provisioner, selected as a member of the voting committee, on the validity of a candidate block. It is used during the [Reduction](./reduction/) phase;
- $\mathsf{Agreement}$: this message contains the aggregated votes of a Reduction phase. It is used at the end of a Reduction phase, if a quorum is reached;
- $\mathsf{AggrAgreement}$: it contains an aggregation of $\mathsf{Agreement}$ messages. It is used in the in the [Ratification](./ratification) phase, if a quorum of such messages is received.

Formally, we denote a consensus message $\mathsf{M}$ as:

$`$\mathsf{M} = (\mathsf{H_M}, \sigma_\mathsf{M}, f_1,\dots,f_n),$`$

where $`\mathsf{H_M}`$ is the message header (defined below), $`\sigma_\mathsf{M}`$ is the provisioner signature on the header, and $f_1, \dots, f_n$ are the other fields specific to each message type.

In the following, we describe both the header and signature in detail.

### Message Header
All consensus messages share a common $MessageHeader$ structure, defined as follows:

| Field       | Type    | Size      | Description                      |
|-------------|---------|-----------|----------------------------------|
| $Signer$    | BLS Key | 96 bytes  | Public Key of the message signer |
| $Round$     | Integer | 64 bits   | Consensus round                  |
| $Step$      | Integer | 8 bits    | Consensus step                   |
| $BlockHash$ | Sha3    | 32 bytes  | Candidate block hash             |

$Signer$ is the public BLS key of the provisioner who creates and signs the message. Therefore, in a $NewBlock$ message, this field identifies the block generator, while, in a $\mathsf{Reduction}$ message, it identifies the committee member who casted the vote.

The $MessageHeader$ structure has a total size of 137 bytes.

### Message Hash and Signature
A *message signature* is the signature of the sender provisioner over the message header's hash.

Formally, given a message $\mathsf{M}$, we define its *hash* as:

$$\eta_\mathsf{M} = Hash_{Blake2B}(Round_\mathsf{M}||Step_\mathsf{M}||BlockHash_\mathsf{M}),$$

where $Round_\mathsf{M}$, $Step_\mathsf{M}$, and $BlockHash_\mathsf{M}$ are the respective fields included in $\mathsf{M}$'s header.

We then define the $\mathsf{M}$'s signature $\sigma_\mathsf{M}$ as:
$$\sigma_\mathsf{M} = Sign_{BLS}(\eta_\mathsf{M}, sk),$$
where $\eta_\mathsf{M}$ is the hash of $\mathsf{M}$'s header and $sk$ is the secret key of the message $Signer_\mathsf{M}$.

The signature $\sigma_\mathsf{M}$ is contained in the $Signature$ field of the message.

With respect to message signatures, we also define the following signing and verification functions:
 - $Sign(\mathsf{M}, sk)$, which takes a message $\mathsf{M}$, a secret key $sk$, and outputs the signature $\sigma_\mathsf{M}$;
 - $Verify(\mathsf{M})$, which takes a message $\mathsf{M}$ and outputs $true$ if $Signature = \sigma_{\mathsf{M}}$, and $false$ otherwise.
 

> Note that the hash operation is actually included in the definition of BLS signature. We explicitly show it here to make it clear and show the actual hash function used in our protocol (Blake2B).


### Message Creation
In addition, we define a common $Msg$ function, which is used to create a consensus message:

$Msg(\mathsf{Type}, f_1,\dots,f_2):$
1. $`\mathsf{H_M} = (pk_\mathcal{\mathcal{N}}, r, s, \mathsf{B}^c)`$
2. $`\sigma_{M} = Sign_{BLS}(\eta_\mathsf{M}, sk_\mathcal{N})`$
3. $`\mathsf{M}^\mathsf{Type} = (\mathsf{H_M}, \sigma_\mathsf{M}, f_1, \dots, f_n)`$
4. $\texttt{output } \mathsf{M}^\mathsf{Type}$

The function uses the local (to the node) consensus parameters and secret key to generate the message header and signature and then build the message with the other specific fields.

$\mathsf{Type}$ indicates the actual message ($\mathsf{NewBlock}$, $\mathsf{Reduction}$, or $\mathsf{Agreement}$).
In case of $\mathsf{Reduction}$, the parameter $f_1$, i.e. the vote $v$, is assigned to $BlockHash$ in the header.


### Message Exchange
While the underlying network protocol is described in the [Network](../network) section, we here define some generic network functions that will be used in the consensus procedures.

***Broadcast***
The *Broadcast* function is used by the creator of a new message to initiate the network propagation of the message. The function simply takes any consensus message in input and is assumed to not fail (only within the context of this description) so it does not return anything.

***Receive***
We use the *Receive* function to process incoming messages. Specifically, we define the *Receive*$(Type,r,s)$ function, which returns a message $\mathsf{M}$ of type $Type$ if it was received from the network and it has $Round_\mathsf{M}=r$ and $Step_\mathsf{M}=s$. If no new message has been received, it returns $NIL$.

Messages received before calling the *Receive* function are stored in a queue and are returned by the function in the order they were received.

***Propagate***
The *Propagate* function represents a re-broadcast operation. It is used by a node when receiving a message from the network and propagating to other nodes.


### Agreement Message
The $\mathsf{Agreement}$ message is used at the end of a successful SA iteration to communicate to the network that a quorum of (positive) votes has been reached. The message is generated by each node participating in the second Reduction committee that collects a quorum of votes for both reductions. The payload includes the $StepVotes$ from the Reduction phases.

| Field       | Type                | Size      | Description                      |
|-------------|---------------------|-----------|----------------------------------|
| $Header$    | [MessageHeader][mh] | 137 bytes | Consensus header                 |
| $Signature$ | BLS Signature       | 48 bytes  | Message signature                |
| $RVotes$    | [StepVotes][sv][ ]  | 112 bytes | First and second Reduction votes |

The $\mathsf{Agreement}$ message has a total size of 297 bytes.


## Consensus Parameters
Consensus parameters are common values that are used throughout the whole consensus protocol. These values are shared by all SA procedures.

Parameters are divided into:
- *global parameters*: network-wide parameters used by all nodes of the network using a particular protocol version;
- *chain state*: represent the current system state, as per result of the execution of all transactions in the blockchain;
- *round state*: local variables used to handle the consensus state 

Additionally, we denote the node running the protocol with $\mathcal{N}$ and refer to its provisioner[^4] keys as $sk_\mathcal{N}$ and $pk_\mathcal{N}$.

**Global Parameters**
<!-- TODO: check when MaxBlockTime is removed -->

All global values (except for the genesis block) refer to version $0$ of the protocol.

| Name               | Value          | Description                                     |
|--------------------|----------------|-------------------------------------------------|
| $GenesisBlock$     | $\mathsf{B_0}$ | Genesis block of the network                    |
| $Version$          | 0              | Protocol version number                         |
| $CommitteeCredits$ | 64             | Total credits in a voting committee             |
| $Quorum$           | 43             | Quorum threshold ($CommitteeCredits \times \frac{2}{3}$) |
| $MaxIterations$    | 71             | Maximum number of iterations for a single round |
| $InitTimeout$      | 5              | Initial step timeout (in seconds)               |
| $MaxTimeout$       | 60             | Maximum timeout for a single step (in seconds)  |
| $MaxBlockTime$     | 360 seconds    | Maximum time to produce a block                 |
| $Dusk$             | 1.000.000.000  | Value of one unit of Dusk (in lux)              |
| $MinStake$         | 1.000          | Minimum amount of a single stake (in Dusk)      |
| $Epoch$            | 2160           | Epoch duration in number of blocks              |
| $BlockGas$         | 5.000.000.000  | Gas limit for a single block                    |


**Chain State**
| Name                 | Description                             |
|----------------------|-----------------------------------------|
| $Tip$                | Current chain tip (last block)          |
| $State$              | Current system state                    |
| $Provisioners$       | Current set of (eligible) provisioners  |

**Round State**
| Name                 | Description                          |
|----------------------|--------------------------------------|
| $Round_{SA}$         | Round number                         |
| $Iteration_{SA}$     | Iteration number                     |
| $\mathsf{B}^c$       | Candidate block                      |
| $\mathsf{B}^w$       | Winning block                        |
| $\mathsf{V}^1$       | StepVotes of First Reduction         |
| $\mathsf{V}^2$       | StepVotes of Second Reduction        |
| $\tau_{Attestation}$ | Current timeout for Attestation      |
| $\tau_{Reduction_1}$ | Current timeout for First Reduction  |
| $\tau_{Reduction_2}$ | Current timeout for Second Reduction |


<!-- $\tau_{Attestation}$, $\tau_{Reduction_1}$, and $\tau_{Reduction_2}$ are all initially set to $InitTimeout$, but might increase in case the timeout expires during an iteration (see [*IncreaseTimeout*](#increasetimeout)). -->

## SA Algorithm
The SA consensus algorithm is defined by the [*SAConsensus*](#saconsensus) procedure, which executes an SA round ([*SARound*](#saround)) for each new block to generate. In turn, the *SARound* procedure runs a loop of [*SAIteration*](#saiteration)s in parallel with the *Ratification* procedure. As soon as a winning block ($\mathsf{B}^w$) for the round is produced, the state is updated ([*StateTransition*](#statetransition)), the $Tip$ is set to the winning block, and a new round begins.

### SAConsensus
The *SAConsensus* procedure is the entry point of a consensus node. Upon booting, the node check if there is a local state saved and, if so, loads it. Otherwise, it starts from the *Genesis Block*. Then, it probes the network to check if it is in sync or not with the blockchain. If not, it starts a synchronization procedure. When in sync, it executes the consensus algorithm per rounds. At the end of each round, if a winning block has been produced, it updates the state and starts a new round. 

If, at any time, the node falls behind the network, the synchronization procedure is run before restarting the rounds loop.

***Algorithm***

1. Load local state
2. If there is a saved state
   1. Check state validity
   2. Set $State$ to loaded state
   3. Set $Tip$ to last block
3. Otherwise
   1. Set $Tip$ to Genesis Block
4. Loop:
   1. Check sync state
   2. If in sync
      1. Set $Round_{SA}$ to $Tip$'s height plus one
      2. Execute Round $Round_{SA}$ to produce winning block $\mathsf{B}^w$
      3. If no winning block has been produced, halt
      4. Execute state transition
   3. Otherwise
      1. Synchronize

***Procedure***

$\textit{SAConsensus}()$
1. $S =$ *LoadState*$()$
2. $\texttt{if } (S \ne NIL):$
   1. *ValidateState*$(S)$
   2. $State = S.State$
   3. $Tip = S.Tip$
3. $\texttt{else}:$
   1. $Tip = GenesisBlock$
4. $\texttt{loop}:$
   1. $inSync =$ *CheckSync*$()$
   2. $\texttt{if } (inSync = true):$
      1. $Round_{SA} = Tip.Height + 1$
      2. $\mathsf{B}^w =$ [*SARound*](#saround)$(Round_{SA})$
      3. $\texttt{if } (\mathsf{B}^w = NIL):$ *Stop*$()$
      4. $State =$ [*StateTransition*](#statetransition)$(\mathsf{B}^w)$
   3. $\texttt{else}:$
      1. *SyncChain*$()$


### SARound
The *SARound* procedure handles the execution of a consensus round: first, it initializes the *Round State* variables; then, it starts the Ratification process in background, and starts executing SA iterations. If, at any time, a winning block is produced by the Ratification process, the round stops.

***Algorithm***

1. Set variables:
   - Initialize Attestation and Reduction timeouts
   - Set candidate and winning block to $NIL$
   - Set iteration to 1
2. Start Ratification process
3. While iteration number is less than $MaxIterations$ and no winning block has been produced
   1. Execute SA iteration
4. If we reached $MaxIterations$ without a winning block
   1. Output $NIL$
5. Otherwise, broadcast the winning block <!-- TODO: move this to SAConsensus ? -->
6. Output the block

***Procedure***

$\textit{SARound}(Round_{SA}):$
1. $\texttt{set }$:
   - $\tau_{Attestation}, \tau_{Reduction_1}, \tau_{Reduction_2} = InitTimeout$
   - $\mathsf{B}^c, \mathsf{B}^w = NIL$
   - $Iteration_{SA} = 1$
2. [*Ratification*][rat]$(Round_{SA})$
3. $\texttt{while } (\mathsf{B}^w = NIL) \texttt{ and } (Iteration_{SA} < MaxIterations$)
   1. [*SAIteration*](#saiteration)$(Round_{SA}, Iteration_{SA})$
4. $\texttt{if } (\mathsf{B}^w = NIL)$
   1. $\texttt{output } NIL$
5. [*Broadcast*][mx]$(\mathsf{B}^w)$
6. $\texttt{output } \mathsf{B}^w$


### SAIteration
This procedure executes a sequence of *Attestation*, to generate a new candidate block ($\mathsf{B}^c$) for the current round and iteration, and two *Reduction* steps, to vote on the candidate block (if any). Quorum votes of first and second Reduction (if any) are stored in $\mathsf{V}^1$ and $\mathsf{V}^2$, respectively.

***Algorithm***
1. Run Attestation to generate *candidate* block $\mathsf{B}^c$
2. Run first Reduction on $\mathsf{B}^c$
3. Run second Reduction on $\mathsf{B}^c$
4. If both Reduction votes are not $NIL$ and this node $\mathcal{N}$ is in the second Reduction committee:
   1. Create $\mathsf{Agreement}$ message $\mathsf{M}^A$ with both Reduction votes
   2. Broadcast message $\mathsf{M}^A$

***Procedure***

$SAIteration(Round, Iteration)$
- $r2Step = (Iteration{-}1) \times 3 + 3$
- $C^{R2} =$ [*DS*][dsa]$(Round,r2Step,CommitteeCredits)$
1. $\mathsf{B}^c =$ [*Attestation*][att]$(Round, Iteration)$
2. $\mathsf{V}^1 =$ [*Reduction*][red]$(Round, Iteration, 1, \mathsf{B}^c)$
3. $\mathsf{V}^2 =$ [*Reduction*][red]$(Round, Iteration, 2, \mathsf{B}^c)$
4. $\texttt{if } (pk_\mathcal{N} \in C^{R2})$ \
   $\texttt{and } (\mathsf{V}^1 \ne NIL) \texttt{ and } (\mathsf{V}^2 \ne NIL):$
    1. $\mathsf{M}^A =$ [*Msg*][msg]$(\mathsf{Agreement}, [\mathsf{V}^1,\mathsf{V}^2])$
        | Field       | Value                         | 
        |-------------|-------------------------------|
        | $Header$    | $\mathsf{H}_\mathsf{M}$       |
        | $Signature$ | $\sigma_\mathsf{M}$           |
        | $RVotes$    | $[\mathsf{V}^1,\mathsf{V}^2]$ |

    2. [*Broadcast*][mx]$(\mathsf{M}^A)$

### StateTransition
*StateTransition* updates the system $State$ by executing the transactions in the winning block $\mathsf{B}^w$, which includes the $Provisioners$ set, and sets the block as the new $Tip$. If agreement was reached in the first iteration, the block is set as *final*.

***Algorithm***

1. Extract $Transactions$, $GasLimit$, and $Generator$ from winning block $\mathsf{B}^w$
2. Generate new state ($newState$) by applying $Transactions$ on the current $State$, and assigning the block reward to $Generator$
3. Update $Provisioners$ set
4. Output $newState$

***Procedure***

$\textit{StateTransition}(\mathsf{B}^w)$
1. $\texttt{set }$:
   - $\boldsymbol{txs} = \mathsf{B}^w.Transactions$
   - $gas = \mathsf{B}^w.GasLimit$
   - $pk_{\mathcal{G}} = \mathsf{B}^w.Generator$
2. $newState =$ *ExecuteTransactions*$(State, \boldsymbol{txs}, gas, pk_{\mathcal{G}})$
3. $Provisioners = newState.Provisioners$
4. $\texttt{if } (Iteration = 1):$ *MakeFinal*$(\mathsf{B}^w)$
5. $Tip = \mathsf{B}^w$

## Synchronization
TBD
<!-- TODO 
SyncChain, CheckSync
Sync when accepting block from the network
  - check header
  - check certificate 
-->

## Common Subroutines
The procedures defined here are common subroutines used throughout the consensus protocol.

### CheckBlockHeader
*CheckBlockHeader* returns $true$ if all block header fields are valid with respect to the previous block and the included transactions. If so, it outputs $true$, otherwise, it outputs $false$.

***Parameters***
- $\mathsf{B}$: block to verify
- $\mathsf{B}^p$: previous block

***Algorithm***
1. Check $Version$ is $0$
2. Check $Height$ is $\mathsf{B}^p$'s height plus 1
3. Check $Hash$ is the header's hash
4. Check $PrevBlock$ is $\mathsf{B}^p$'s hash
5. Check $Timestamp$ is higher than $\mathsf{B}^p$'s timestamp
6. Check $Timestamp$ is not higher than $\mathsf{B}^p$'s timestamp plus $MaxBlockTime$
7. Check transaction root is correct with respect to the transaction set
8. Check state hash corresponds to the result of the state transition over $\mathsf{B}^p$
- If all checks passed
  1. Output $true$
- Otherwise
  1. Output $false$

***Procedure***

$CheckBlockHeader(\mathsf{B}, \mathsf{B}^p)$:
- $newState =$ *ExecuteTransactions*$(State_{\mathsf{B}^p}, \mathsf{B}.Transactions), BlockGas, pk_{G_\mathsf{B}})$
- $\texttt{if }$
  1. $(\mathsf{B}.Version = 0)$ 
  2. $\texttt{and } (\mathsf{B}.Height = \mathsf{B}^p.Height)$
  3. $\texttt{and } (\mathsf{B}.Hash =$ *Hash*$`_{SHA3}(\mathsf{H}_{\mathsf{B}}))`$
  4. $\texttt{and } (\mathsf{B}.PreviousBlock = \mathsf{B}^p.Hash)$
  5. $\texttt{and } (\mathsf{B}.Timestamp \ge \mathsf{B}^p.Timestamp)$
  6. $\texttt{and } (\mathsf{B}.Timestamp \le \mathsf{B}^p.Timestamp + MaxBlockTime)$
  7. $\texttt{and } (\mathsf{B}.TransactionRoot = MerkleTree(\mathsf{B}.Transactions).Root)$
  8. $\texttt{and } (\mathsf{B}^c.StateRoot = newState.Root):$
     1. $\texttt{output } true$
- $\texttt{else}:$
   1. $\texttt{output } false$

### VerifyCertificate
<!-- TODO : VerifyCertificate-->
TBD

***Parameters***
- $\mathsf{B}$: block
- $\mathsf{C}$: certificate

$CheckCertificate(\mathsf{B},\mathsf{C})$

### IncreaseTimeout
*IncreaseTimeout* doubles a step timeout up to $MaxTimeout$.

***Parameters***
- $\tau_{Step}$: $Step$ timeout to increase (where $Step$ can be $Attestation$, $Reduction1$, or $Reduction2$)

***Procedure***

$\textit{IncreaseTimeout}()$
- $\tau_{Step} =$ *Max*$(\tau_{Step} \times 2, MaxTimeout)$

<!----------------------- FOOTNOTES ----------------------->

[^1]: A type of Proof-of-Stake consensus mechanism that relies on a committee of validators, rather than all validators in the network, to reach consensus on the next block. Committee-based PoS mechanisms often have faster block times and lower overhead than their non-committee counterparts, but may also be more susceptible to censorship or centralization.

[^2]: A finality guarantee that is achieved through the accumulation of blocks over time, such that the probability of a block being reversed decreases exponentially as more blocks are added on top of it. This type of guarantee is in contrast to absolute finality, which is achieved when it is mathematically impossible for a block to be reversed.

[^3]: In principle, a malicious block generator could create two valid candidate blocks. However, this case is automatically handled in the Reduction phase, since provisioners will reach agreement on a specific block hash.

[^4]: For the sake of simplicity, we assume the node handle a single provisioner identity.

<!------------------------- LINKS ------------------------->

[cp]: #consensus-parameters
[ds]: ./sortition/
[dsa]: ./sortition/README.md#deterministic-sortition-ds
[att]: ./attestation/README.md#attestation-algorithm
[red]: ./reduction/README.md#reduction-algorithm
[sv]: ./reduction/README.md#stepvotes
[rat]: ./ratification/README.md#ratification-algorithm
[msg]: #message-creation
[mx]: #message-exchange
[mh]: #message-header
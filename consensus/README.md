<!-- TODO: mention ProcessBlock -->
<!-- TODO rename Consensus Parameters to Environment -->
<!-- TODO: All nodes can contribute to consensus by collecting votes and generating an Agreement message as soon as a quorum is reached on both Reductions -->
<!-- TODO: Define sth like Candidate "signature": block_hash, round, iteration -->
# Succinct Attestation
**Succinct Attestation** (**SA**) is a permissionless, committee-based Proof-of-Stake consensus protocol. 

The protocol is run by Dusk stakers, known as ***provisioners***, which are responsible for generating and validating new blocks.

Provisioners participate in turns to the production and validation of each new block of the ledger. Participation in each round is decided with a [*Deterministic Sortition*][ds] (*DS*) algorithm, which is used to extract a unique *block generator* and unique *voting committees* among provisioners, in a decentralized, non-interactive way.

### ToC
  - [Notation](#notation)
    - [Procedure Execution](#procedure-execution)
  - [Protocol Overview](#protocol-overview)
    - [Candidate Block](#candidate-block)
  - [Provisioners and Stakes](#provisioners-and-stakes)
    - [Epochs and Eligibility](#epochs-and-eligibility)
  - [Certificates](#certificates)
    - [Structures](#structures)
      - [Certificate](#certificate)
      - [StepVotes](#stepvotes)
  - [Consensus Parameters](#consensus-parameters)
  - [SA Algorithm](#sa-algorithm)
    - [SAConsensus](#saconsensus)
    - [SALoop](#saloop)
    - [SARound](#saround)
    - [SAIteration](#saiteration)
    - [IncreaseTimeout](#increasetimeout)


## Notation
<!-- TODO: replace \mathsf for `` so structure names can be embedded in links -->
In this documentation we will use the following notation:

- $Latex$ is used for:
  - variables, with lowercase letters (e.g. a variable $v$)
  - simple objects, with uppercase letters (e.g. a committee $C$)
  - consensus parameters, state variables, and object fields, with whole words, (e.g. the $Quorum$ parameter)
- $\boldsymbol{Bold}$ is used for vectors (e.g. a bitset $\boldsymbol{bs}$)
- $\mathcal{Calligrafic}$ is used for actors (e.g. a provisioner $\mathcal{P}$)
- $\mathsf{Sans Serif}$ is used for structures (e.g. a block $\mathsf{B}$, or a $\mathsf{Reduction}$ message)
- *Italic* is used for function names (e.g. the *Hash* function)

Moreover, we will conventionally use the following symbols:
- $pk$ and $sk$ are used to denote public and secret keys, respectively
- $\sigma$ characters are used for signatures (e.g. a message signature $\sigma_{M}$)
- $\eta$ characters are used for hash digests (e.g. a block hash $\eta_B$)
- $\tau$ characters are used for time variables (e.g. the Proposal timeout $\tau_{Proposal}$)
- $\mathcal{G}$ is used to the block generator
- $\mathsf{B}^c$ is used for the candidate block
- $\mathsf{B}^w$ is used for the winning block

While we can use the dot notation to specify a structure field (e.g. $\mathsf{B}.Header$), we will prefer the use of subscript when convenient (e.g. $\mathsf{H_B} = \mathsf{B}.Header$). Generally speaking, we use subscripts to specify the object or actor to which a variable belongs to (e.g. a block's hash $\eta_\mathsf{B}$), to indicate numeric order (e.g. the $i\text{th}$ block $`\mathsf{B}_i`$), or to specify a particular instantiation (e.g. *Hash*$`_{BLS}`$). In the case of vectors, we also use the standard notation $[i]$ to indicate the $i\text{th}$ element.

When using the dot notation, we will omit intermediate fields when possible. For instance, we will use $\mathsf{B}.Seed$ to denote $\mathsf{B}.Header.Seed$

We use $\leftarrow$ to assign object fields to separate variables. For instance, $\mathsf{H_B}, \boldsymbol{txs} \leftarrow \mathsf{B}$ will assign $\mathsf{B}.Header$ to $\mathsf{H_B}$, and $B.Transactions$ to $\boldsymbol{txs}$. The assignation follows the field order in the related structure definition. If a field is not needed in the algorithm, we ignore it by assigning it to a null variable $`\_`$.

### Procedure Execution
Some procedures (typically containing a loop) are executed concurrently to each other (e.g., as parallel threads). We have these procedures be controlled using the following commands:
- $\texttt{start}(P)$: initiates the procedure $P$;
- $\texttt{stop}(P)$: interrupts the procedure $P$;
- $\texttt{restart}(P)$: stops and restart the procedure $P$.

Within these procedures, we use the command $\texttt{stop}$ (with no arguments) to indicate the interruption of the execution.

Additionally, we define the function $\texttt{running}(P)$ which outputs $true$ if $P$ is currently running, and $false$ otherwise.

## Protocol Overview
The SA protocol is executed in ***rounds***, with each round adding a new block to the chain.

Each round is composed of three phases:
  1. ***Proposal***: in this phase, a *generator*, extracted via [*DS*][ds], creates a new candidate block and broadcasts it to the network using a [Candidate][cmsg] message;
  
  2. ***Reduction***: in this phase, two separate committees of provisioners, extracted via [*DS*][ds], vote on the validity of the candidate block and broadcast their vote with a [Reduction][rmsg] message. If positive votes reach a quorum of $\frac{2}{3}$ (67% of the votes) in each committee, an $\mathsf{Agreement}$ message is broadcast, which contains the aggregated votes of both committees. Conversely, the reduction fails and the protocol starts over (from *Proposal*) with a new generator and new voting committees (that is, a new candidate block is produced and voted upon);

  3. ***Ratification***: in this phase, all generated $\mathsf{Agreement}$ messages are collected; if messages for a specific candidate block reach a quorum (according to the second-reduction committee vote allocation), the candidate block is added to the chain along with a *certificate* containing the quorum votes from the $\mathsf{Agreement}$ message. This effectively ends the current round.

As both the *Proposal* and *Reduction* phases can fail, the *Proposal*/*Reduction* sequence can be executed multiple times within a single round (using different provisioners). We call each repetition of such sequence an ***iteration***, and each phase in the repetition a ***step***.
Note that since the *Reduction* phase has two consecutive votes, it is composed of two steps. Therefore, a single iteration is composed of three steps: (1) Proposal, (2) first Reduction, and (3) second Reduction.

A maximum number of 71 iterations (213 steps) is allowed to agree on a new valid block to add to the chain. If this number is exceeded, the network is considered unsafe and the consensus process is halted. Both step and iteration count starts from 1. 

### Candidate Block
A candidate block is the block generated in the [Proposal][prop] step by the provisioner extracted as block generator. This is the block on which other provisioners will have to reach an agreement. If an agreement is not reached by the end of the iteration, a new candidate block will be produced and a new iteration will start.

Therefore, for each iteration, only one (valid) candidate block can be produced[^1]. To reflect this, we denote a candidate block with $\mathsf{B}^c_{r,i}$, where $r$ is the consensus round, and $i$ is the consensus iteration. 
Note that we simplify this notation to simply $\mathsf{B}^c$ when this does not generate confusion.

A candidate block that reaches an agreement is called a *winning* block.

## Provisioners and Stakes
A provisioner is a user that locks a certain amount of their Dusk coins as *stake* (see [*Stake Contract*][c-stake]).
Formally, we define a *Provisioner* $\mathcal{P}$ as:

$$\mathcal{P}=(pk_\mathcal{P}, S_\mathcal{P}),$$

where $pk_\mathcal{P}$ is the BLS public key of the provisioner and $S_\mathcal{P}$ is the stake belonging to $\mathcal{P}$.

In turn, a *Stake* is defined as:
$$S_\mathcal{P}=(Amount, Height),$$
where $\mathcal{P}$ is the provisioner that owns the stake, $Amount$ is the quantity of locked Dusks, and $Height$ is the height of the block where the lock action took place (i.e., when the *stake* transaction was included). The minimum value for a stake is defined by the [global parameter][cp] $MinStake$, and is currently equivalent to 1000 Dusk.

The stake amount of each provisioner directly influences the probability of being extracted by the [Deterministic Sortition][dsa] algorithm: the higher the stake, the more the provisioner will be extracted on average.

### Epochs and Eligibility
Participation in the consensus protocol is marked by *epochs*. Each epoch corresponds to a fixed number of blocks, defined by the [global parameter][cp] $Epoch$ (currently equal to 2160).
<!-- TODO: why 2160 ? -->

The *Provisioners Set* during an epoch is fixed: all `stake` and `unstake` operations take effect at the beginning of an epoch. We refer to this property as *epoch stability*.

Additionally, for a Provisioner to be included in the Provisioners List, its stake is required to wait a certain number of blocks known as the *maturity* period. After this period, the stake and the Provisioner are said to be *eligible* for Sortition.

Formally, we say a Stake $S$ is *eligible* at round $R$, if $R$ is at least $Maturity$ blocks higher than the round in which $S$ was staked:

$$`R \ge Round_S + M,`$$

where $Round_S$ is the round at which $S$ was staked, and $M$ is the *maturity* period defined as:

$$`M = 2{\times}Epoch - (Round_S \mod Epoch)),`$$

where $Epoch$ is a [global parameter][cp]. Note that the value of $M$ is equal to a full epoch plus the blocks from $Round_S$ to the end of the corresponding epoch[^2]. Therefore the value of $M$ will vary depending on $Round_S$:

$$`Epoch \lt M \le 2{\times}Epoch.`$$

Having the Provisioner Set fixed during an epoch, along with the use of the maturity period, is mainly intended to allow the pre-verification of blocks from the future: if a node falls behind the main chain and receives a block at a higher height than its tip, it can pre-verify this block by ensuring that the block generator and the committee members are part of the expected Provisioner List.
Specifically, while the stability of the Provisioner List during an epoch allows to pre-verify blocks from the same epoch, the Maturity period allows to also pre-verify blocks from the next epoch, since all changes (stake/unstake) occurred between the tip and the end of the epoch will not be effective until the end of the following epoch.

## Certificates
<!-- TODO: Rename to Attestation and define Certificate as the PrevBlock Cert -->
A *Certificate* is an aggregated vote from a quorum committee along with the bitset of the voters. It is used as proof of a committee reaching Quorum or NilQuorum in the Reduction steps. 
In particular, Quorum Certificates are used to prove a candidate block reached consensus and can be accepted to the chain; on the other hand, NilQuorum Certificates are used to prove that a candidate failed to reach a quorum (and can't reach any).

### Structures
#### Certificate
The $\mathsf{Certificate}$ structure contains the aggregated votes of [sub-committees][sc] of the two [Reduction][rmsg] steps that reached quorum on a candidate block.
It is composed of two $\mathsf{StepVotes}$ structures, one for each [Reduction][red] step.


| Field             | Type            | Size     | Description                          |
|-------------------|-----------------|----------|--------------------------------------|
| $FirstReduction$  | [StepVotes][sv] | 56 bytes | Aggregated votes of first reduction  |
| $SecondReduction$ | [StepVotes][sv] | 56 bytes | Aggregated votes of second reduction |

The $\mathsf{Certificate}$ structure has a total size of 112 bytes.

#### StepVotes
The $\mathsf{StepVotes}$ structure is used to store votes from the [Validation][val] and [Ratification][rat] steps.
Votes are aggregated BLS signatures from members of a Voting Committee. Each vote is a signature for a triplet $(round, step, vote)$.
To specify which committee members' votes are included, a [sub-committee bitset][bs] is used.

The structure is defined as follows:

| Field    | Type          | Size     | Description                                |
|----------|---------------|----------|--------------------------------------------|
| $Voters$ | BitSet        | 64 bits  | Bitset of the voters                       |
| $Votes$  | BLS Signature | 48 bytes | Aggregated $\mathsf{Reduction}$ signatures |

The $\mathsf{StepVotes}$ structure has a total size of 56 bytes.

### Procedures
#### *AggregateVote*
*AggregateVote* adds a vote to a [StepVotes][sv] by aggregating the BLS signature and setting the signer bit in the committee bitset.

**Parameters**
- $\mathsf{SV}$: the $\mathsf{StepVotes}$ structure with the aggregated votes
- $\mathsf{C}$: the Voting Committee of $\mathsf{SV}$
- $\sigma$: the BLS signature to aggregate
- $pk$: the signer public key

**Procedure**

$\textit{AggregateVote}( \mathsf{SV}, \mathsf{C}, \sigma, pk ) :$
1. $\mathsf{SV}.Votes =$ *BLS_Aggregate*$(\mathsf{SV}.Votes, \sigma)$
2. $\mathsf{SV}.Voters = $ [*SetBit*][sb]$(\mathsf{SV}.Voters, \mathsf{C}, pk)$
3. $\texttt{output } \mathsf{SV}$

## Consensus Parameters
<!-- Rename to Environment -->
Consensus parameters are common values that are used throughout the whole consensus protocol. These values are shared by all SA procedures.

Parameters are divided into:
- *global parameters*: network-wide parameters used by all nodes of the network using a particular protocol version;
- *chain state*: represent the current system state, as per result of the execution of all transactions in the blockchain;
- *round state*: local variables used to handle the consensus state 

Additionally, we denote the node running the protocol with $\mathcal{N}$ and refer to its provisioner[^3] keys as $sk_\mathcal{N}$ and $pk_\mathcal{N}$.

**Global Parameters**
<!-- TODO: check when MaxBlockTime is removed -->

All global values (except for the genesis block) refer to version $0$ of the protocol.

| Name                    | Value          | Description                                         |
|-------------------------|----------------|-----------------------------------------------------|
| $GenesisBlock$          | $\mathsf{B_0}$ | Genesis block of the network                        |
| $Version$               | 0              | Protocol version number                             |
| $CommitteeCredits$      | 64             | Total credits in a voting committee                 |
| $Quorum$                | $CommitteeCredits \times \frac{2}{3}$ (43) | Quorum threshold        |
| $NilQuorum$             | $CommitteeCredits - Quorum +1$ (22) | Quorum threshold for NIL votes |
| $MaxIterations$         | 71             | Maximum number of iterations for a single round     |
| $RollingFinalityBlocks$ | 5              | Number of Attested blocks for [Rolling Finality][rf] |
| $InitTimeout$           | 5              | Initial step timeout (in seconds)                   |
| $MaxTimeout$            | 60             | Maximum timeout for a single step (in seconds)      |
| $MaxBlockTime$          | 360            | Maximum time to produce a block (in seconds)        |
| $Dusk$                  | 1.000.000.000  | Value of one unit of Dusk (in lux)                  |
| $MinStake$              | 1.000          | Minimum amount of a single stake (in Dusk)          |
| $Epoch$                 | 2160           | Epoch duration in number of blocks                  |
| $BlockGas$              | 5.000.000.000  | Gas limit for a single block                        |


**Chain State**
<!-- TODO: Add Type column -->
| Name                 | Description                             |
|----------------------|-----------------------------------------|
| $Tip$                | Current chain tip (last block)          |
| $State$              | Current system state                    |
| $Provisioners$       | Current set of (eligible) provisioners  |

**Round State**
| Name                          | Description                          |
|-------------------------------|--------------------------------------|
| $Round_{SA}$                  | Round number                         |
| $Iteration_{SA}$              | Iteration number                     |
| $\mathsf{B}^c$                | Candidate block                      |
| $\mathsf{B}^w$                | Winning block                        |
| $\mathsf{V}^1$                | StepVotes of First Reduction         |
| $\mathsf{V}^2$                | StepVotes of Second Reduction        |
| $\tau_{Proposal}$          | Current timeout for Proposal      |
| $\tau_{Reduction_1}$          | Current timeout for First Reduction  |
| $\tau_{Reduction_2}$          | Current timeout for Second Reduction |
| $\boldsymbol{PrevIterations}$ | Certificates of failed iterations    |


<!-- $\tau_{Proposal}$, $\tau_{Reduction_1}$, and $\tau_{Reduction_2}$ are all initially set to $InitTimeout$, but might increase in case the timeout expires during an iteration (see [*IncreaseTimeout*](#increasetimeout)). -->

## SA Algorithm
The SA consensus algorithm is defined by the [*SAConsensus*][sac] procedure, which executes an SA round ([*SARound*][sar]) for each new block to generate. In turn, the *SARound* procedure runs a loop of [*SAIteration*][sai]s in parallel with the *Ratification* procedure. As soon as a winning block ($\mathsf{B}^w$) for the round is produced, the state is updated ([*AcceptBlock*][ab]), the $Tip$ is set to the winning block, and a new round begins.

### SAConsensus
The *SAConsensus* procedure is the entry point of a consensus node. Upon booting, the node checks if there is a local state saved and, if so, loads it. Otherwise, it starts from the *Genesis Block*. Then, it probes the network to check if it is in sync or not with the blockchain. If not, it starts a synchronization procedure. When in sync, it executes the consensus algorithm per rounds. At the end of each round, if a winning block has been produced, it updates the state and starts a new round. 

If, at any time, the node falls behind the network, the synchronization procedure is run before restarting the rounds loop.

***Algorithm***

1. Load local state
2. If there is no saved state
   1. Set $Tip$ to Genesis Block
3. Otherwise
   1. Check state validity
   2. Set $State$ to loaded state
   3. Set $Tip$ to last block
4. Start SA loop procedure ([*SALoop*][sal])

***Procedure***

$\textit{SAConsensus}():$
1. $S =$ *LoadState*$()$
2. $\texttt{if } (S = NIL):$
   1. $Tip = GenesisBlock$
3. $\texttt{else}:$
   1. *ValidateState*$(S)$    <!-- TODO -->
   2. $State = S.State$
   3. $Tip = S.Tip$
4. $\texttt{start}$([*SALoop*][sal])

### SALoop
The *SALoop* procedure executes SA rounds (*SARound*). We define this as a separate procedure to allow it to be stopped and restarted when synchronizing the blockchain.

The procedure executes an infinite loop of SA rounds. At each iteration, the $Round_{SA}$ global variable is set to $Tip$'s height plus one; then the SA round is executed; if the round produced a winning block, it executes the state transition using this block. If any round ends with no winning block, the consensus process is considered as faulty and halts. Suc an event requires a manual recovery procedure.

***Algorithm***

1. Loop:
   1. Set $Round_{SA}$ to $Tip$'s height plus one
   2. Execute Round $Round_{SA}$ to produce winning block $\mathsf{B}^w$
   3. If no winning block has been produced, halt consensus
   4. Execute state transition

***Procedure***

$\textit{SALoop}():$
1. $\texttt{loop}:$
   1. $Round_{SA} = Tip.Height + 1$
   2. $\mathsf{B}^w =$ [*SARound*][sar]$(Round_{SA})$
   3. $\texttt{if } (\mathsf{B}^w = NIL):$ 
       - $\texttt{stop}$
   4. $State =$ [*AcceptBlock*][ab]$(\mathsf{B}^w)$

### SARound
The *SARound* procedure handles the execution of a consensus round: first, it initializes the *Round State* variables; then, it starts the Ratification process in background, and starts executing SA iterations. If, at any time, a winning block is produced by the Ratification process, the round stops.

***Algorithm***

1. Set variables:
   - Initialize Proposal and Reduction timeouts
   - Set candidate and winning block to $NIL$
   - Set iteration to 0
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
   - $\tau_{Proposal}, \tau_{Reduction_1}, \tau_{Reduction_2} = InitTimeout$
   - $\mathsf{B}^c, \mathsf{B}^w = NIL$
   - $Iteration_{SA} = 0$
2. $\texttt{start}$([*Ratification*][rata]$(Round_{SA}))$
3. $\texttt{while } (\mathsf{B}^w = NIL) \texttt{ and } (Iteration_{SA} \le MaxIterations)$
   1. [*SAIteration*][sai]$(Round_{SA}, Iteration_{SA})$
4. $\texttt{if } (\mathsf{B}^w = NIL)$
   1. $\texttt{output } NIL$
5. [*Broadcast*][mx]$(\mathsf{B}^w)$
6. $\texttt{output } \mathsf{B}^w$


### SAIteration
This procedure executes a sequence of *Proposal*, to generate a new candidate block ($\mathsf{B}^c$) for the current round and iteration, and two *Reduction* steps, to vote on the candidate block (if any). Quorum votes of first and second Reduction (if any) are stored in $\mathsf{V}^1$ and $\mathsf{V}^2$, respectively. If the votes of both committees reach a quorum, an [Agreement][amsg] message broadcasted, containing all quorum votes.


***Algorithm***
1. Run Proposal to generate *candidate* block $\mathsf{B}^c$
2. Run first Reduction on $\mathsf{B}^c$
3. Run second Reduction on $\mathsf{B}^c$
4. If any of the two Reductions failed
   1. Create a certificate with the aggregated votes
   2. Add the failed-iteration certificate to $\boldsymbol{PrevIterations}$
5. and this node $\mathcal{N}$ is in the second Reduction committee:
   1. Create $\mathsf{Agreement}$ message $\mathsf{M}^A$ with both Reduction votes
   2. Broadcast message $\mathsf{M}^A$

***Procedure***

$\textit{SAIteration}(Round, Iteration):$
- $r2Step = Iteration\times 3 + 2$
- $C^{R2} =$ [*DS*][dsa]$(Round,r2Step,CommitteeCredits)$
1. $\mathsf{B}^c =$ [*Proposal*][ps]$(Round, Iteration)$
2. $(v^1, \mathsf{V}^1) =$ [*Reduction*][reda]$(Round, Iteration, 1, \mathsf{B}^c)$
3. $(v^2, \mathsf{V}^2) =$ [*Reduction*][reda]$(Round, Iteration, 2, \mathsf{B}^c)$
4. $\texttt{if } (v^1 = NIL) \texttt{ or } (v^1 = NIL)$
   1. $\mathsf{C} = {\mathsf{V}^1, \mathsf{V}^2}$
   2. $\boldsymbol{PrevIterations}[Iteration] = {\mathsf{C}}$
5. $\texttt{else if} (pk_\mathcal{N} \in C^{R2}):$
    1. $\mathsf{M}^A =$ [*Msg*][msg]$(\mathsf{Agreement}, [\mathsf{V}^1,\mathsf{V}^2])$
        | Field       | Value                         | 
        |-------------|-------------------------------|
        | $Header$    | $\mathsf{H}_\mathsf{M}$       |
        | $Signature$ | $\sigma_\mathsf{M}$           |
        | $RVotes$    | $[\mathsf{V}^1,\mathsf{V}^2]$ |

    2. [*Broadcast*][mx]$(\mathsf{M}^A)$


### IncreaseTimeout
*IncreaseTimeout* doubles a step timeout up to $MaxTimeout$.

***Parameters***
- $\tau_{Step}$: $Step$ timeout to increase (where $Step$ can be $Proposal$, $Reduction1$, or $Reduction2$)

***Procedure***

$\textit{IncreaseTimeout}(\tau_{Step}):$
- $\tau_{Step} =$ *Max*$(\tau_{Step} \times 2, MaxTimeout)$

<!-- TODO: Define $GetStepNumber$ 
Proposal: (Block.Iteration) \times 3 + 1
red1: (Block.Iteration) \times 3 + 1
red2: (Block.Iteration) \times 3 + 2
-->

<!----------------------- FOOTNOTES ----------------------->

[^1]: In principle, a malicious block generator could create two valid candidate blocks. However, this case is automatically handled in the Reduction phase, since provisioners will reach agreement on a specific block hash.

[^2]: Note that an epoch refers to a specific set of blocks and not just to a number of blocks; that is, an epoch starts and ends at specific block heights.

[^3]: For the sake of simplicity, we assume the node handles a single provisioner identity.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md -->
[sa]:   #protocol-overview
[cert]: #certificates
[sv]:   #stepvotes
[av]:   #aggregatevote
[cp]:   #consensus-parameters
[sac]:  #saconsensus
[sar]:  #saround
[sai]:  #saiteration
[sal]:  #saloop
[it]:   #increasetimeout

[net]: https://github.com/dusk-network/dusk-protocol/tree/main/network

<!-- Chain Management -->
[ab]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#acceptblock
[pb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#processblock
[rf]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#rolling-finality
<!-- Sortition -->
[ds]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/
[dsa]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#algorithm
[sc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#subcommittee
[sb]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#setbit
<!-- Proposal -->
[prop]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal/
[ps]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal/README.md#proposalstep
<!-- Reduction -->
[red]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/
[reda]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md#reduction-algorithm
<!-- Ratification -->
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/
[rata]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md#ratification-algorithm
<!-- Messages -->
[msg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-creation
[mx]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-exchange
[mh]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-header
[amsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#agreement-message
[rmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#reduction-message
[cmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#candidate-message

<!-- TODO: link to Stake Contract -->
[c-stake]: https://github.com/dusk-network/dusk-protocol/tree/main/contracts/stake
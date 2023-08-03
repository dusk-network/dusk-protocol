<!-- TODO: Describe main consensus loop -->
<!-- TODO: Define Provisioners and stake -->
<!-- TODO: Define State := current blockchain -->
<!-- TODO: mention the algorithm assumes the running node only contains a single provisioner key, although it can be easily modified to handle multiple keys -->
<!-- TODO: rename NewBlock to Candidate -->
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
- $\mathbf{Bold}$ is used for vectors (e.g. a bitset $\mathbf{bs}$)
- $\mathcal{Calligrafic}$ is used for actors (e.g. a provisioner $\mathcal{P}$)
- $\mathsf{Sans Serif}$ is used for structures (e.g. a block $\mathsf{B}$, or a $\mathsf{Reduction}$ message)
- *Italic* is used for functions names (e.g. the *Hash* function)
- $pk$ and $sk$ are used to denote public and secret keys, respectively
- $\sigma$ characters are used for signatures (e.g. a message signature $\sigma_{M}$)
- $\eta$ characters are used for hash digests (e.g. a block hash $\eta_B$)
- $\tau$ characters are used for time variables (e.g. the Attestation timeout $\tau_{Attestation}$)

While we can use the dot notation to specify a structure field (e.g. $\mathsf{B}.Header$), we will prefer the use of subscript when convenient (e.g. $\mathsf{H_B} = \mathsf{B}.Header$). Generally speaking, we use subscripts to specify the object or actor to which a variable belongs to (e.g. a block's hash $\eta_\mathsf{B}$), to indicate numeric order (e.g. the $i\text{th}$ block $`\mathsf{B}_i`$), or to specify a particular instantiation (e.g. *Hash*$`_{BLS}`$). In the case of vectors, we also use the standard notation $[i]$ to indicate the $i\text{th}$ element.

We use $\leftarrow$ to assign object fields to separate variables. For instance, $\mathsf{H_B}, \textbf{txs} \leftarrow \mathsf{B}$ will assign $\mathsf{B}.Header$ to $\mathsf{H_B}$, and $B.Transactions$ to $\textbf{txs}$. The assignation follows the field order in the related structure definition. If a field is not needed in the algorithm, we ignore it by assigning it to a null variable $`\_`$.

## Protocol Overview
The SA protocol is executed in ***rounds***, with each round adding a new block to the chain.

Each round is composed of three phases:
  1. ***Attestation***: in this phase, a *generator*, extracted via [*DS*][ds], creates a new candidate block and broadcasts it to the network using a $\mathsf{NewBlock}$ message;
  
  2. ***Reduction***: in this phase, two separate committees of provisioners, extracted via [*DS*][ds], vote on the validity of the candidate block and broadcast their vote with a $\mathsf{Reduction}$ message. If positive votes reach a quorum of $\frac{2}{3}$ (67% of the votes) in each committee, an $\mathsf{Agreement}$ message is broadcast, which contains the aggregated votes of both committees. Conversely, the reduction fails and the protocol starts over (from *Attestation*) with a new generator and new voting committees (that is, a new candidate block is produced and voted upon);

  3. ***Ratification***: in this phase, all generated $\mathsf{Agreement}$ messages are collected; if messages for a specific candidate block reach a quorum (according to the second-reduction committee vote allocation), the candidate block is added to the chain along with a *certificate* containing the quorum votes from the $\mathsf{Agreement}$ message. This effectively ends the current round.

As the *Reduction* phase can fail, multiple repetitions of the *Attestation*/*Reduction* sequence can occur within a single round. We call each such repetition an ***iteration***, and each phase in the repetition a ***step***.
Note that since the *Reduction* phase has two consecutive votes, it is composed of two steps. Therefore, a single iteration is composed of three steps: (1) Attestation, (2) first Reduction, and (3) second Reduction.

A maximum number of 71 iterations (213 steps) is allowed to add a new valid block to the chain. If this number is exceeded, the network is deemed unsafe and the consensus process is halted.


## Provisioners and Stakes
<!-- TODO: link to Stake Contract -->
<!-- TODO: mention minimum amount of stake -->
A provisioner is a user that locked a certain amount of their Dusk coins as *stake* (see [*Stake Contract*]()).

Formally, we define a *provisioner* ${P}$ as:

$${P}=(pk_P,\mathbf{Stakes}_P),$$

where $pk_P$ is the BLS public key of the provisioner, and $\mathbf{Stakes}_P = [S_0,\dots,S_n]$, where $S_i$ is a stake amount belonging to ${P}$.

In turn, a *stake* is defined as:
$$S^{P}=(Amount, Height),$$
where ${P}$ is the provisioner that owns the stake, $Amount$ is the quantity of locked Dusks, and $Height$ is the height of the block where the lock action took place (i.e., when the *stake* transaction was included).

We say a stake $S$ is *mature* if it was locked more than two *epochs* before the current chain tip. Formally, given a stake $S$, we say $S$ is mature if 

$$Height_{Tip} \gt Height_S + (2 \times Epoch),$$

where $Height_{Tip}$ is the current tip's height, $Height_S$ is the stake's height, and $Epoch$ is a global [Consensus Parameter][cp]. 

Moreover, we say a provisioner is *eligible* if it has at least one mature stake.

Note that only eligible provisioners participate in the consensus protocol.


<!-- TODO: mv Voting Committees here? -->

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
<!-- TODO: mv $Signer$ outside the header -->
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

$Msg(Type, f_1,\dots,f_2):$
1. $`\mathsf{H_M} = (pk_\mathcal{\mathcal{N}}, r, s, \mathsf{B}^c)`$
2. $`\sigma_{M} = Sign_{BLS}(\eta_\mathsf{M}, sk_\mathcal{N})`$
3. $`\mathsf{M} = (\mathsf{H_M}, \sigma_\mathsf{M}, f_1, \dots, f_n)`$
4. $\texttt{output } \mathsf{M}$

The function uses the local (to the node) consensus parameters and secret key to generate the message header and signature and then build the message with the other specific fields.

$Type$ indicate the actual message ($\mathsf{NewBlock}$, $\mathsf{Reduction}$, or $\mathsf{Agreement}$).
In case of $\mathsf{Reduction}$, the parameter $f_1$, i.e. the vote $v$, is assigned to $BlockHash$ in the header.


#### Message Exchange
While the underlying network protocol is described in the [relative section](../network), we here define some generic network functions that will be used in the consensus procedures.

**Broadcast**
The *Broadcast* function is used by the creator of a new message to initiate the network propagation of the message. The function simply takes any consensus message in input and is assumed to not fail (only within the context of this description) so it does not return anything.

**Receive**
We use the *Receive* function to process incoming messages. Specifically, we define the *Receive*$(Type,r,s)$ function, which returns a message $\mathsf{M}$ of type $Type$ if it was received from the network and it has $Round_\mathsf{M}=r$ and $Step_\mathsf{M}=s$. If no new message has been received, it returns $NIL$.

Messages received before calling the *Receive* function are stored in a queue and are returned by the function in the order they were received.

**Propagate**
The *Propagate* function represents a re-broadcast operation. It is used by a node when receiving a message from the network and propagating to other nodes.

## Consensus Parameters
<!-- DOING -->
Consensus parameters are common values used throughout the whole consensus protocol. They are divided into *global parameters* and *state variables*. Global parameters are network-wide parameters used by all nodes of the network using a particular protocol version. Context variables are local to each node and are used to both identify the node's provisioner and handle the local consensus state.

**Global Parameters**
| Name                | Value         | Description                                    |
|---------------------|---------------|------------------------------------------------|
| $CommitteeCredits$            | 64            | Total credits in a voting committee            |
| $Quorum$            | 43            | Quorum threshold ($CommitteeCredits \times \frac{2}{3}$) |
| $MaxSteps$          | 213           | Maximum number of steps for a single round     |
| $InitTimeout$       | 5             | Initial step timeout (in seconds)              |
| $MaxTimeout$ | 60            | Maximum timeout for a single step (in seconds) |
| $Epoch$             | 2160          | Epoch duration in number of blocks             |
| $Dusk$              | 1.000.000.000 | Value of one unit of Dusk (in lux)             |
| $BlockGas$ | 5.000.000.000 | Gas limit for a single block                   |
| $MaxTxSetSize$      | 825000        | Maximum size of transaction set in a block     |

<!-- TODO: MaxTxSetSize is never used (here). check in the code -->
<!-- TODO: Motivate MaxTxSetSize = 825000  -->
<!-- Will be removed
| **`MaxBlockTime`**      | 360 seconds   | ?                             | 
-->

**State Variables**
<!-- TODO: replace $Step_{SA}$ with $Iteration$ ? -->
| Name                 | Description                          |
|----------------------|--------------------------------------|
| $\mathcal{N}$        | Node (provisioner) running the protocol            |
| $pk_\mathcal{N}$     | PubKey of the node provisioner       |
| $Version$            | Protocol version number              |
| $Tip$                | Current chain tip (last block)        |
| $Round_{SA}$         | Current consensus round              | <!-- TODO: replace $r$ with $Round$ -->
| $Step_{SA}$          | Current consensus step               | <!-- TODO: replace $s$ with $Step$ ? -->
| $\tau_{Attestation}$ | Current timeout for Attestation      |
| $\tau_{Reduction_1}$ | Current timeout for First Reduction  |
| $\tau_{Reduction_2}$ | Current timeout for Second Reduction |

$\tau_{Attestation}$, $\tau_{Reduction_1}$, and $\tau_{Reduction_2}$ are all initially set to $InitTimeout$, but might increase in case the timeout expires during an iteration (see [*IncreaseTimeout*](#increasetimeout)).

<!-- TODO: Add "Initial value", if not included in main loop algorithm -->

## Consensus Algorithm
<!-- TODO -->
### SAIteration
<!-- TODO NewIteration -->

## Common Subroutines
#### IncreaseTimeout
`IncreaseTimeout` increases a step timeout up to the maximum step timeout ($MaxTimeout$).

Input: $\tau_{Step}$

Procedure:
- If $\tau_{Step} \times 2 < MaxTimeout$
  - $\tau_{Step} = \tau_{Step} \times 2$
- Else
  - $\tau_{Step} = MaxTimeout$


<!-------------------- Footnotes -------------------->

[^1]: A type of Proof-of-Stake consensus mechanism that relies on a committee of validators, rather than all validators in the network, to reach consensus on the next block. Committee-based PoS mechanisms often have faster block times and lower overhead than their non-committee counterparts, but may also be more susceptible to censorship or centralization.

[^2]: A finality guarantee that is achieved through the accumulation of blocks over time, such that the probability of a block being reversed decreases exponentially as more blocks are added on top of it. This type of guarantee is in contrast to absolute finality, which is achieved when it is mathematically impossible for a block to be reversed.


<!-- LINKS -->
[cp]: #consensus-parameters
[ds]: ./sortition/

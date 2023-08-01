<!-- TODO: Describe main consensus loop -->
<!-- TODO: Define Provisioners and stake -->
<!-- TODO: Define State := current blockchain -->
<!-- TODO: mention the algorithm assumes the running node only contains a single provisioner key, although it can be easily modified to handle multiple keys -->
# Succinct Attestation
**Succinct Attestation** (**SA**) is a permissionless, committee-based[^1] Proof-of-Stake consensus protocol that provides statistical finality guarantees[^2]. 

The protocol is run by Dusk stakers, known as ***provisioners***, which are responsible for generating, validating, and finalizing new blocks.

Provisioners participate in turns to the production and validation of each new block of the ledger. Participation in each round is decided with a [*Deterministic Sortition*][ds] algorithm, which is used to extract a unique *block generator* and unique *voting committees* among provisioners, in a decentralized, non-interactive way.


## Protocol Overview
The SA protocol is executed in ***rounds***, with each round adding a new block to the chain.

Each round is composed of three phases:
  1. ***Attestation***: in this phase, a *generator*, extracted via [DS][ds], creates a new candidate block and broadcasts it to the network using a $NewBlock$ message;
  
  2. ***Reduction***: in this phase, two separate committees of provisioners, extracted via [DS][ds], vote on the validity of the candidate block and broadcast their vote with a $Reduction$ message. If positive votes reach a quorum of $\frac{2}{3}$ (67% of the votes) in each committee, an $Agreement$ message is broadcast, which contains the aggregated votes of both committees. Conversely, the reduction fails and the protocol starts over (from *Attestation*) with a new generator and new voting committees (that is, a new candidate block is produced and voted upon);

  3. ***Ratification***: in this phase, all generated $Agreement$ messages are collected; if messages for a specific candidate block reach a quorum (according to the second-reduction committee vote allocation), the candidate block is added to the chain along with a *certificate* containing the quorum votes from the $Agreement$ message. This effectively ends the current round.

As the *Reduction* phase can fail, multiple repetitions of the *Attestation*/*Reduction* sequence can occur within a single round. We call each such repetition an ***iteration***, and each phase in the repetition a ***step***.
Note that since the *Reduction* phase has two consecutive votes, it is composed of two steps. Therefore, a single iteration is composed of three steps: (1) Attestation, (2) first Reduction, and (3) second Reduction.

A maximum number of 71 iterations (213 steps) is allowed to add a new valid block to the chain. If this number is exceeded, the network is deemed unsafe and the consensus process is halted.


## Provisioners and Stakes
<!-- TODO: link to Stake Contract -->
<!-- TODO: mention minimum amount of stake -->
A provisioner is a user that locked a certain amount of their DUSK coins as *stake* (see [*Stake Contract*]()).

Formally, we define a ***provisioner*** $\mathcal{P}$ as:

$$\mathcal{P}=(pk,[S_0,\dots,S_n]),$$

where $pk$ is the BLS public key of the provisioner, and $S_i$ is a stake amount belonging to $\mathcal{P}$.

In turn, a ***stake*** is defined as:
$$S^\mathcal{P}=(Amount, Height),$$
where $\mathcal{P}$ is the provisioner that owns the stake, $Amount$ is the quantity of locked Dusks, and $Height$ is the height of the block where the lock action took place (i.e., when the *stake* transaction was included).

We say a stake $S$ is ***mature*** if it was locked more than two *epochs* before the current chain tip. Formally, given a stake $S$, we say $S$ is mature if 

$$Height_{Tip} \gt Height_S + (2 \times EPOCH),$$

where $Height_{Tip}$ is the current tip's height, $Height_S$ is the stake's height, and $EPOCH$ is a global [Consensus Parameter][cp]. 

Moreover, we say a provisioner is ***eligible*** if it has at least one mature stake.

Note that only eligible provisioners participate in the consensus protocol.


<!-- TODO: mv Voting Committees here? -->

## Consensus Messages
To run the SA protocol, participating nodes exchange consensus messages. There are four types of message:
- $NewBlock$: this message stores a candidate block for a certain round and iteration. It is used during the [Attestation](./attestation/) phase;
- $Reduction$: this message contains the vote of a provisioner, selected as a member of the voting committee, on the validity of a candidate block. It is used during the [Reduction](./reduction/) phase;
- $Agreement$: this message contains the aggregated votes of a Reduction phase. It is used at the end of a Reduction phase, if a quorum is reached;
- $AggrAgreement$: it contains an aggregation of $Agreement$ messages. It is used in the in the [Ratification](./ratification) phase, if a quorum of such messages is received.

Formally, we denote a consensus message $\mathcal{M}$ as:

$`$\mathcal{M} = (\mathcal{H}_\mathcal{M}, \sigma_\mathcal{M}, f_1,\dots,f_n),$`$
<!-- $$\mathsf{m} = (\mathsf{h}_\mathsf{m}, \sigma_\mathsf{m}, f_1,\dots,f_n),$$
 -->

where $`\mathcal{H}_\mathcal{M}`$ is the message header (defined below), $`\sigma_\mathcal{M}`$ is the provisioner signature on the header, and $f_1, \dots, f_n$ are the other fields specific to each message type.

In the following, we describe both the header and signature in detail.

### Message Header
<!-- TODO: mv $Signer$ outside the header -->
All consensus messages share a common $MessageHeader$ structure, defined as follows:

| Field       | Type     | Size      | Description                       |
|-------------|----------|-----------|-----------------------------------|
| $Signer$    | BLS Key  | 96 bytes  | Public Key of the message signer  |
| $Round$     | Integer  | 64 bits   | Consensus round                   |
| $Step$      | Integer  | 8 bits    | Consensus step                    |
| $BlockHash$ | Sha3-256 | 32 bytes  | Candidate block hash              |

$Signer$ is the public BLS key of the provisioner who creates and signs the message. Therefore, in a $NewBlock$ message, this field identifies the block generator, while, in a $Reduction$ message, it identifies the committee member who casted the vote.

The $MessageHeader$ structure has a total size of 137 bytes.

### Message Hash and Signature
A *message signature* is the signature of the sender provisioner over the message header's hash.

Formally, given a message $\mathcal{M}$, we define its *hash* as:

$$\eta_\mathcal{M} = H_{Blake2B}(Round_\mathcal{M}||Step_\mathcal{M}||BlockHash_\mathcal{M}),$$

where $Round_\mathcal{M}$, $Step_\mathcal{M}$, and $BlockHash_\mathcal{M}$ are the respective fields included in $\mathcal{M}$'s header.

We then define the $\mathcal{M}$'s signature $\sigma_{\mathcal{M}}$ as:
$$\sigma_{\mathcal{M}} = Sig_{BLS}(\eta_\mathcal{M}, sk),$$
where $\eta_\mathcal{M}$ is the hash of $\mathcal{M}$'s header and $sk$ is the secret key of the message $Signer_\mathcal{M}$.

The signature $\sigma_{\mathcal{M}}$ is contained in the $Signature$ field of the message.

With respect to message signatures, we also define the following signing and verification functions:
 - $Sign(\mathcal{M}, sk)$, which takes a message $\mathcal{M}$, a secret key $sk$, and outputs the signature $\sigma_{\mathcal{M}}$;
 - $Verify(\mathcal{M})$, which takes a message $\mathcal{M}$ and outputs $true$ if $Signature = \sigma_{\mathcal{M}}$, and $false$ otherwise.
 

> Note that the hash operation is actually included in the definition of BLS signature. We explicitly show it here to make it clear and show the actual hash function used in our protocol (Blake2B).


### Message Creation
In addition, we define a common $Msg$ function, which is used to create a consensus message:

$Msg(Type, f_1,\dots,f_2):$
1. $`\mathcal{H}_\mathcal{M} = (pk_\mathcal{\mathcal{N}}, r, s, \mathcal{B}^c)`$
2. $`\sigma_{\mathcal{M}} = Sig_{BLS}(\eta_\mathcal{M}, sk_\mathcal{N})`$
3. $`\mathcal{M} = (\mathcal{H}_\mathcal{M}, \sigma_{\mathcal{M}}, f_1, \dots, f_n)`$
4. $output \text{ } \mathcal{M}$

The function uses the local (to the node) consensus parameters and secret key to generate the message header and signature and then build the message with the other specific fields.

$Type$ indicate the actual message ($NewBlock$, $Reduction$, or $Agreement$).
In case of $Reduction$, the parameter $f_1$, i.e. the vote $v$, is assigned to $BlockHash$ in the header.


#### Message Exchange
While the underlying network protocol is described in the [relative section](../network), we here define some generic network functions that will be used in the consensus procedures.

**Broadcast**
The *Broadcast* function is used by the creator of a new message to initiate the network propagation of the message. The function simply takes any consensus message in input and is assumed to not fail (only within the context of this description) so it does not return anything.

**Receive**
We use the $Receive$ function to process incoming messages. Specifically, we define the *Receive*$(Type,r,s)$ function, which returns a message $\mathcal{M}$ of type $Type$ if it was received from the network and it has $Round_\mathcal{M}=r$ and $Step_\mathcal{M}=s$. If no new message has been received, it returns $NIL$.

Messages received before calling the *Receive* function are stored in a queue and are returned by the function in the order they were received.

**Propagate**
The *Propagate* function represents a re-broadcast operation. It is used by a node when receiving a message from the network and propagating to other nodes.

## Consensus Parameters
<!-- DOING -->
Several parameters are used in the SA procedures.
We divide them into _configuration constants_, which are network-wide parameters, and _context variables_, which are specific to the running node and its state current state with respect to the SA protocol.

**Config Constants**
| Name                  | Value         | Description                                    |
|-----------------------|---------------|------------------------------------------------|
| **`DUSK`**            | 1.000.000.000 | Value of 1 Dusk unit (in lux)                  |
| $Gas^{\mathcal{B}}$   | 5.000.000.000 | Gas limit for a single block                   |
| $\tau_{Step}$         | 5 seconds     | Initial step timeout                           |
| $\tau_{Step}^{max}$   | 60 seconds    | Maximum timeout for a single step              | <!-- **`MaxStepTimeout`** -->
| **`ExtractionDelay`** | 3 seconds     | Extra delay to fetch transactions from mempool |
| **`MaxTxSetSize`**    | 825000        | Maximum size of transaction set in a block     |
| $VCPool$              | 64            | Total credits in a voting committee            |
| $Quorum$              | 43            | Quorum threshold ($VCPool \times \frac{2}{3}$) |
| $MaxSteps$            | 213           | Maximum number of steps for a single round     |
| $EPOCH$               | 2160          | Epoch duration in number of blocks             |


<!-- TODO: Motivate MaxTxSetSize = 825000  -->
<!-- Will be removed
| **`MaxBlockTime`**      | 360 seconds   | ?                             | 
-->

**Context Variables**
| Name                  | Description                          |
|-----------------------|--------------------------------------|
| $v$                   | Protocol version number              |
| $\mathcal{N}$                   | Node running the protocol            |
| $pk_\mathcal{N}$                | PubKey of the node provisioner       |
| $\mathcal{B_{tip}}$   | Current chain tip (last block)       |
| $r$                   | Current consensus round              |
| $s$                   | Current consensus step               | <!-- TODO: replace $s$ with iteration $i$ ? -->
| $\tau_{Attestation}$  | Current timeout for Attestation      |
| $\tau_{Reduction_1}$  | Current timeout for First Reduction  |
| $\tau_{Reduction_2}$  | Current timeout for Second Reduction |

<!-- This are included in the previous block
| **`prevHash`**          | Previous block's hash       |
| **`prevTimestamp`**     | Previous block's timestamp  |
| **`prevSeed`**          | Previous block's seed       | -->


$\tau_{Attestation}$, $\tau_{Reduction_1}$, and $\tau_{Reduction_2}$ are all initially set to $\tau_{Step}$, but might increase in case the timeout expires during an iteration (see [*IncreaseTimeout*](#increasetimeout)).



## Consensus Algorithm
<!-- TODO -->
### SAIteration
<!-- TODO NewIteration -->

## Common Subroutines
#### IncreaseTimeout
`IncreaseTimeout` increases a step timeout up to `MaxStepTimeout`.

Input: $StepTimeout$

Procedure:
- If $StepTimeout \times 2 < MaxStepTimeout$
  - $StepTimeout = StepTimeout \times 2$
- Else
  - $StepTimeout = MaxStepTimeout$


## Notation
- $\mathcal{H}^\mathcal{B}$ denotes the hash of the header of block $\mathcal{B}$.
- The notation $H_{SHA3}$ indicates the SHA3-256 hash function.

<!-------------------- Footnotes -------------------->

[^1]: A type of Proof-of-Stake consensus mechanism that relies on a committee of validators, rather than all validators in the network, to reach consensus on the next block. Committee-based PoS mechanisms often have faster block times and lower overhead than their non-committee counterparts, but may also be more susceptible to censorship or centralization.

[^2]: A finality guarantee that is achieved through the accumulation of blocks over time, such that the probability of a block being reversed decreases exponentially as more blocks are added on top of it. This type of guarantee is in contrast to absolute finality, which is achieved when it is mathematically impossible for a block to be reversed.

<!-- OLD -->

<!-- ```
Provisioner {
  byte[]    PubKey,       // The Provisioner ID
  int       StakeWeight,  // The total amount of mature stake
  Stake[]   Stakes        // Set of stakes
}

Stake {
  int Amount,             // The amount staked
  int BlockHeight         // The block when the amount was staked
}
``` -->

<!-- TODO: notation 
- Signatures: \sigma
- hashes: \eta
- Structures: \mathcal
- Arrays: \bold

$(\mathcal{H}_\mathcal{M},\_,\mathcal{B}_r^i,\sigma_\mathcal{M}) \leftarrow \mathcal{M}$
- We use the concise notation to declare variables from structure fields.
- We borrow the use of _ to indicate an unused field

We use $\tau_{Now}$ to denote the current time in UNIX format.
-->

<!-- LINKS -->
[cp]: #consensus-parameters
[ds]: ./sortition/

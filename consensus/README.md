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


## Provisioners
*Provisioners* are the only users in the Dusk network that are allowed to participate in the SA protocol. A provisioner is a user that locked a certain amount of their DUSK coins as *stake*.

Staking is managed through the *Stake Contract*. 
<!-- TODO: link to Stake Contract -->
<!-- TODO: mention minimum amount of stake -->

Formally, we define a ***provisioner*** $\mathcal{P}$ as:

$$\mathcal{P}=(pk,[S_0,\dots,S_n]),$$

where $pk$ is the BLS public key of the provisioner, and $S_i$ is a stake amount belonging to $\mathcal{P}$.

In turn, we define a ***stake*** $S$ of a provisioner $\mathcal{P}$ as:
$$S^\mathcal{P}=(Amount,Height),$$
where $Amount$ is the quantity of Dusk locked as a stake, and $Height$ is the height of the block where $Amount$ was locked.

We say a stake $S$ is ***mature*** if it was locked more than two *epochs* before the current chain tip. Formally, given a stake $S$, we say $S$ is mature if 

$$Height_{Tip} \gt Height_S + (2 \times EPOCH),$$

where $Height_{Tip}$ is the current tip's height, $Height_S$ is the stake's height, and $EPOCH$ is a [Consensus Parameter][cp]. 

Moreover, we say a provisioner is ***eligible*** if it has at least a mature stake.

Note that only eligible provisioners can participate in the consensus protocol.

<!-- Formally:
  $$\mathcal{P}^e = \mathcal{P} : \exists \text{ } S^\mathcal{P} : S^\mathcal{P} \gt Height_{Tip} \gt Height_S + (2 \times EPOCH)$$ -->


<!-- TODO: decide if $P$ only include mature stakes -->


<!-- TODO: mv Voting Committees here? -->

## Protocol Messages
The SA protocol is executed by nodes by means of message exchange. There are four types of message exchanged during the consensus protocol:
- $NewBlock$: produced in the [Attestation](./attestation/) phase, it stores a candidate block for the current round;
- $Reduction$: used during the [Reduction](./reduction/) phase, it contains a single vote from a committee member on a candidate block;
- $Agreement$: produced at the end of the Reduction phase in case a quorum is reached, it contains the aggregated votes of the two reduction steps;
- $AggrAgreement$: produced in the [Ratification](./ratification) phase, it contains the aggregated signature and bitset of the votes confirming the candidate block.

We denote a consensus message $\mathcal{M}$ as:

$`$\mathcal{M} = (\mathcal{H}_\mathcal{M}, \sigma_\mathcal{M}, f_1,\dots,f_n),$`$
<!-- $$\mathsf{m} = (\mathsf{h}_\mathsf{m}, \sigma_\mathsf{m}, f_1,\dots,f_n),$$
 -->

where $`\mathcal{H}_\mathcal{M}`$ is the message header, $`\sigma_\mathcal{M}`$ is the signature of the sender, and $f_1, \dots, f_n$ are the fields specific to the message type.

In the following, we describe both the header and signature in detail.

### Message Header
All consensus messages (with the exception of $AggrAgreement$, which embeds an $Agreement$ message) share a common $MessageHeader$ structure, defined as follows:

| Field       | Type    | Size      | Description                       |
|-------------|---------|-----------|-----------------------------------|
| $Signer$    | BLS Key | 96 bytes  | Public Key of the message signer  |
| $Round$     | Number  | 64 bits   | Consensus round                   |
| $Step$      | Number  | 8 bits    | Consensus step                    |
| $BlockHash$ | Hash    | 32 bytes  | Candidate block hash              |

$Signer$ is the public BLS key of the Provisioner who created and signed the message. For instance, in a $NewBlock$ message, this field indicates the block generator, while, in a $Reduction$ message, it indicates the committee member who casted the vote.

The $MessageHeader$ structure has a total size of 137 bytes.

#### Message Signature
<!-- TODO: mv pk away from header -->
When a Provisioner creates a consensus message (except for $AggrAgreement$), it signs its header for authenticity and integrity. 

In particular, given a message $\mathcal{M}$, we define its hash as:

$$\eta_\mathcal{M} = H_{Blake2B}(Round||Step||BlockHash),$$

where $Round$, $Step$, and $BlockHash$ are the respective fields included in $\mathcal{M}$'s header.

We then define the $\mathcal{M}$'s signature $\sigma_{\mathcal{M}}$ as:
$$\sigma_{\mathcal{M}} = Sig_{BLS}(\eta_\mathcal{M}, sk),$$
where $\eta_\mathcal{M}$ is the hash of $\mathcal{M}$'s header and $sk$ is the secret key of the signer.

The signature $\sigma_{\mathcal{M}}$ is then put in the $Signature$ field of the message.

In addition, we define the following message signing and verifying functions:
 - $Sign(\mathcal{M}, sk)$, which takes a message $\mathcal{M}$, a secret key $sk$, and outputs the signature $\sigma_{\mathcal{M}}$;
 - $Verify(\mathcal{M})$, which takes a message $\mathcal{M}$ and outputs $true$ if $\mathcal{M}.Signature$ corresponds to $\sigma_{\mathcal{M}}$, and $false$ otherwise.
 

> Note that the hash operation is actually included in the definition of BLS signature. We explicitly show it here to make it clear and show the actual hash function used in our protocol (Blake2B).


#### Create Message
In addition, we define the $NewMsg$ function, which is used to create a consensus message:

$Msg(Type, f_1,\dots,f_2):$
1. $`\mathcal{H}_\mathcal{M} = (pk_\mathcal{\mathcal{N}}, r, s, \mathcal{B}^c)`$
2. $`\sigma_{\mathcal{M}} = Sig_{BLS}(\eta_\mathcal{M}, sk_\mathcal{N})`$
3. $`\mathcal{M} = (\mathcal{H}_\mathcal{M}, \sigma_{\mathcal{M}}, f_1, \dots, f_n)`$
4. $output \text{ } \mathcal{M}$

$Type$ indicate the actual message ($NewBlock$, $Reduction$, or $Agreement$).
In case of $Reduction$, the parameter $f_1$, i.e. the vote $v$, is assigned to $BlockHash$ in the header.


#### Message Exchange
**Receive**
We handle incoming message with the *Receive*$(MessageType,r,s)$ function, which returns a message $\mathcal{M}$ of type $MessageType$ if it was received and it has $Round=r$ and $Step=s$. If no new message has been received, it returns $NIL$.

Messages received before calling the *Receive* function are stored in a queue and are returned by the function in the order they were received.

**Broadcast**
The *Broadcast* function is used by the creator of a new message.
<!-- TODO: Send -->

**Propagate**
<!-- TODO -->
The *Propagate* function represent a re-broadcast operation. It is used by a node when receiving a message from the network and propagating to other nodes.


## Consensus Parameters
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

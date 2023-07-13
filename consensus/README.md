<!-- TODO: Describe main consensus loop -->
<!-- TODO: Define Provisioners and stake -->
<!-- TODO: Define State := current blockchain -->
<!-- TODO: mention the algorithm assumes the running node only contains a single provisioner key, although it can be easily modified to handle multiple keys -->
# Succinct Attestation
**_Succinct Attestation_ (_SA_)** is a permissionless, committee-based[^1] Proof-of-Stake consensus protocol that provides statistical finality guarantees[^2]. 

The protocol is run by Dusk stakers, called *Provisioners*, in a sequence of *rounds*, each of which produces a new block of the blockchain. 
The protocol leverages the *Deterministic Sortition* algorithm, which allows selecting a unique block producer and unique voting committees in a non-interactive deterministic way.

## Protocol Overview
The SA consensus is divided into **_rounds_**, each of which creates a new block. In turn, each round is composed of one or more **_iterations_** of the following **_phases_**:

  1. **_Attestation_**: in this phase, a _generator_, extracted among eligible Provisioners, creates a new candidate block $B$ for the current round, and broadcasts it to the network;
  
  2. **_1st Reduction_**: in this phase, the members of a _committee_, extracted among the  eligible Provisioners, vote on the validity of the candidate block $B$; 
  if votes reach a quorum of $\frac{2}{3}$ (i.e., 67% of the committee voting pool), the reduction outputs $B$, otherwise it outputs $NIL$;

  3. **_2nd Reduction_**: in this phase, if the output of the 1st Reduction is not $NIL$, a second _committee_, also extracted among the eligible Provisioners, votes on the candidate block $B$;
  if votes reach the quorum, an `Agreement` message is broadcast, which contains all votes of the two Reduction phases.

> NOTE: Iteration phases are also known as **_steps_**, so that each iteration is composed of 3 steps.
<!-- TODO: mention maximum number of steps -->

A successful iteration (i.e., one that produces a valid new block), is ratified by the following phase:
 - **_Ratification_**: in this phase, which runs concurrently to iteration steps, `Agreement` messages for the current round are collected and processed by nodes. If collected votes reach a quorum, the new block is accepted by producing a `Certificate` message with the aggregated votes. This message is then propagated to be verified by other nodes.


## Provisioners
*Provisioners* are the only users in the Dusk network that are allowed to participate in the SA protocol. A provisioner is a user that locked a certain amount of DUSK coins in the *Stake Contract*. <!-- TODO: link to Stake Contract -->

<!-- TODO: mention minimum amount of stake -->

Formally, we define a *stake* $S$ 

we define a *Provisioner* as:

$$P=(pk,[S_0,\dots,S_n]),$$

where $S_i$ a stake belonging to $P$.

In turn, we define a *stake* $S$ as:

$$S^P=(Amount,H),$$

where $Amount$ is the quantity of Dusk locked as a stake,and $H$ is the block height when such $Amount$ was staked.

We say a stake $S$ is *mature* if $H^S\gt 2\times EPOCH$, where $EPOCH$ is a Consensus Parameter, currently set at $?$. <!-- TODO: define epoch -->
We say a provisioner $P$ is *eligible* if it has at least a mature stake.


<!-- TODO: decide if $P$ only include mature stakes -->


## Deterministic Sortition
The $SA$ algorithm leverages [*Deterministic Sortition*](./sortition/) to select a unique block producer during the [*Attestation*](./attestation/) phase, as well as forming unique voting committee during the [Reduction](./reduction/) phase.

## Consensus Parameters
Several parameters are used in the SA procedures.
We divide them into _configuration constants_, which are network-wide parameters, and _context variables_, which are specific to the running node and its state current state with respect to the SA protocol.

**Config Constants**
| Name                    | Value         | Description                   |
|-------------------------|---------------|-------------------------------|
| **`DUSK`**              | 1.000.000.000 | Value of 1 Dusk unit (in lux) |
| $Gas^{\mathcal{B}}$     | 5.000.000.000 | Gas limit for a single block  |
| $\tau_{Step}$  | 5 seconds     | Initial step timeout          |
| $\tau_{Step}^{max}$     | 60 seconds    | Maximum timeout for a single step | <!-- **`MaxStepTimeout`** -->
| **`ExtractionDelay`**   | 3 seconds     | Extra delay to fetch transactions from mempool |
| **`MaxTxSetSize`**      | 825000        | Maximum size of transaction set in a block     |
| $VCPool$                | 64            | Total credits in a voting committee |
| $Quorum$                | 43            | Quorum threshold ($VCPool \times \frac{2}{3}$) |
| $MaxSteps$  | 213           | Maximum number of steps for a single round |

<!-- Will be removed
| **`MaxBlockTime`**      | 360 seconds   | ?                             | 
-->

**Context Variables**
| Name                  | Description                          |
|-----------------------|--------------------------------------|
| $v$                   | Protocol version number              |
| $N$                   | Node running the protocol            |
| $pk_N$                | PubKey of the node provisioner       |
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


## Protocol Messages
<!-- TODO: Receive(type, r, s) -->
There are four types of message exchanged during the consensus protocol:
- $NewBlock$: produced in the [Attestation](./attestation/) phase, it stores a candidate block for the current round;
- $Reduction$: used during the [Reduction](./reduction/) phase, it contains a single vote from a committee member on a candidate block;
- $Agreement$: produced at the end of the Reduction phase in case a quorum is reached, it contains the aggregated votes of the two reduction steps;
- $AggrAgreement$: produced in the [Ratification](./ratification) phase, it contains the aggregated signature and bitset of the votes confirming the candidate block.

### Consensus Message Header
All consensus messages (with the exception of $AggrAgreement$, which embeds an $Agreement$ message) share a common $MessageHeader$ structure, defined as follows:

<!-- TODO: rename PubKeyBLS to pk_BLS and BlockHash to CB -->
| Field     | Type    | Size      | Description                       |
|-----------|---------|-----------|-----------------------------------|
| $Signer$ | BLS Key | 96 bytes  | Public Key of the message signer  |
| $Round$     | Number  | 64 bits   | Consensus round                   |
| $Step$      | Number  | 8 bits    | Consensus step                    |
| $BlockHash$ | Hash    | 32 bytes  | Candidate block hash              |

$Signer$ is the public BLS key of the Provisioner who created and signed the message. As such, in a $NewBlock$ message, this field indicates the block generator, while in a $Reduction$ message it indicates the committee member who casted the vote.

#### Message Signature
<!-- TODO: mv pk away from header -->
When a Provisioner creates a consensus message (except for $AggrAgreement$), it signs its header for authenticity and integrity.

In particular, given a message $\mathcal{M}$, the signature $\sigma_{\mathcal{M}}$ is defined as
$$\sigma_{\mathcal{M}} = Sig_{BLS}(H_{Blake2B}(Round||Step||BlockHash))$$

We define two functions: 
 - $Sign_{BLS}(\mathcal{M})$, which is used to digitally sign the message
 - $Verify_{BLS}(\mathcal{M}, pk)$, which is used to verify a signature on a message

> Note that the hash operation is actually included in the definition of BLS signature. We explicitly show it here to make it clear and show the actual hash function used in our protocol (Blake2B).

## Common Subroutines
#### IncreaseTimeout
`IncreaseTimeout` increases a step timeout up to `MaxStepTimeout`.

Input: $StepTimeout$

Procedure:
- If $StepTimeout \times 2 < MaxStepTimeout$
  - $StepTimeout = StepTimeout \times 2$
- Else
  - $StepTimeout = MaxStepTimeout$




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


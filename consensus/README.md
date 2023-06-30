# Dusk Consensus Protocol
The Dusk blockchain leverages **_Succinct Attestation_ (_SA_)**, which is a permissionless, committee-based[^1] Proof-of-Stake consensus protocol that provides statistical finality guarantees[^2]. 

## Participants
To participate in the SA consensus, a Dusk user must meet the following requirements:
 - to be a **_Provisioner_**, i.e., have a pre-configured amount of DUSK locked as a stake;
 - to be **_Eligible_**, i.e., have a stake with an age of at least two epochs.  <!-- TODO: define epoch -->

To formally describe the consensus protocol, we define the *Provisioner* structure as:
```
Provisioner {
  byte[]    PubKey,       // The Provisioner ID
  int       StakeWeight,  // The total amount of mature stake
  Stake[]   Stakes        // Set of stakes
}

Stake {
  int Amount,             // The amount staked
  int BlockHeight         // The block when the amount was staked
}
```

## Messages
Three message types are exchanged during the consensus protocol:
- `NewBlock`: produced in the [Attestation](./attestation/) phase, it stores a candidate block for the current round;
- `Reduction`: used during the [Reduction](./reduction/) phase, it contains a single vote from a committee member on a candidate block;
- `Agreement`: produced at the end of the Reduction phase in case a quorum is reached, it contains the aggregated votes of the two reduction steps;
- `AggrAgreement`: produced in the [Ratification](./ratification) phase, it contains the aggregated signature and bitset of the votes confirming the candidate block.

### Consensus Message Header
All consensus messages, with the exception of `AggrAgreement` that contains an `Agreement` message, share a common header structure, defined as follows:

| Field     | Type   | Size      | Description                       |
|-----------|--------|-----------|-----------------------------------|
| PubKeyBLS | byte[] | 96 bytes  | Public Key of the message creator |
| Round     | uint   | 64 bits   | Consensus round                   |
| Step      | uint   | 8 bits    | Consensus step                    |
| BlockHash | byte[] | 32 bytes  | Block hash                        |

`PubKeyBLS` is the ID of the Provisioner who creator and signer of the message. So, for instance, in a `NewBlock` message, this field would indicate the ID of the block generator.

## Workflow
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

### Extraction 
The extraction process, used to select the block producer in the Attestation phase and the committees in the Reduction phases, leverages the **_Deterministic Sortition_** algorithm, described [here](sortition/README.md)[^3]. 
<!-- TODO: add link to description -->


### Parameters
Several parameters are used in the SA procedures.
We divide them into _configuration constants_, which are network-wide parameters, and _context variables_, which are specific to the running node and its state current state with respect to the SA protocol.

**Config Constants**
| Name                    | Value         | Description                   |
|-------------------------|---------------|-------------------------------|
| **`DUSK`**              | 1.000.000.000 | Value of 1 Dusk unit (in lux) |
| **`BlockGasLimit`**     | 5.000.000.000 | Gas limit for a single block  |
| **`MaxBlockTime`**      | 360 seconds   | ?                             | <!-- TODO -->
| **`ConsensusTimeOut`**  | 5 seconds     | ?                             | <!-- TODO -->
| **`ExtractionDelay`**   | 3 seconds     | Extra delay to fetch transactions from mempool |
| **`MaxTxSetSize`**      | 825000        | Maximum size of transaction set in a block     |

**Context Variables**
| Name                    | Description                 |
|-------------------------|-----------------------------|
| **`node`**              | Node running the protocol   | <!-- TODO: mention/define its content (keys) -->
| **`round`**             | Current consensus round (i.e., block height) |
| **`prevHash`**          | Previous block's hash       |
| **`prevTimestamp`**     | Previous block's timestamp  |
| **`prevSeed`**          | Previous block's seed       |
| **`queue`**             | Consensus event queue       | <!-- TODO: delete this? -->

<!-------------------- Footnotes -------------------->

[^1]: A type of Proof-of-Stake consensus mechanism that relies on a committee of validators, rather than all validators in the network, to reach consensus on the next block. Committee-based PoS mechanisms often have faster block times and lower overhead than their non-committee counterparts, but may also be more susceptible to censorship or centralization.

[^2]: A finality guarantee that is achieved through the accumulation of blocks over time, such that the probability of a block being reversed decreases exponentially as more blocks are added on top of it. This type of guarantee is in contrast to absolute finality, which is achieved when it is mathematically impossible for a block to be reversed.
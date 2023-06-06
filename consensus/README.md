# Dusk Consensus Protocol
The Dusk blockchain leverages **_Succinct Attestation_ (_SA_)**, which is a permissionless, committee-based[^1] Proof-of-Stake consensus protocol that provides statistical finality guarantees[^2]. 

### Participants
To participate in the SA consensus, a Dusk user must meet the following requirements:
 - to be a **_Provisioner_**, i.e., have a pre-configured amount of DUSK locked as a stake;
 - to be **_Eligible_**, i.e., have a stake with an age of at least two epochs.

### Workflow
The SA consensus is divided into **_rounds_**, each of which creates a new block. In turn, each round is composed of one or more **_iterations_** of the following **_phases_**:

  1. **_Attestation_**: in this phase, a _generator_, extracted among eligible Provisioners, creates a new candidate block $B$ for the current round, and broadcasts it to the network;
  
  2. **_1st Reduction_**: in this phase, the members of a _committee_, extracted among the  eligible Provisioners, vote on the validity of the candidate block $B$; 
  if votes reach a quorum of $2/3$ (i.e., 67% of the committee voting pool), the reduction outputs $B$, otherwise it outputs $NIL$;

  3. **_2nd Reduction_**: in this phase, if the output of the 1st Reduction is not $NIL$, a second _committee_, also extracted among the eligible Provisioners, votes on the candidate block $B$;
  if votes reach the quorum, an `Agreement` message is broadcast, which contains all votes of the two Reduction phases.

> NOTE: Iteration phases are also known as **_steps_**, so that each iteration is composed of 3 steps.
<!-- TODO: mention maximum number of steps -->

A successful iteration (i.e., one that produces a valid new block), is ratified by the following phase:
 - **_Ratification_**: in this phase, which runs concurrently to iteration steps, `Agreement` messages for the current round are collected and processed by nodes. If collected votes reach a quorum, the new block is accepted by producing a `Certicate` message with the aggregated votes. This message is then propagated to be verified by other nodes.

### Extraction 
The extraction process, used to select the block producer in the Attestation phase and the committees in the Reduction phases, leverages the **_Deterministic Sortition_** algorithm, described [here](sortition/README.md)[^3]. 
<!-- TODO: add link to description -->


<!-------------------- Footnotes -------------------->

[^1]: A type of Proof-of-Stake consensus mechanism that relies on a committee of validators, rather than all validators in the network, to reach consensus on the next block. Committee-based PoS mechanisms often have faster block times and lower overhead than their non-committee counterparts, but may also be more susceptible to censorship or centralization.

[^2]: A finality guarantee that is achieved through the accumulation of blocks over time, such that the probability of a block being reversed decreases exponentially as more blocks are added on top of it. This type of guarantee is in contrast to absolute finality, which is achieved when it is mathematically impossible for a block to be reversed.
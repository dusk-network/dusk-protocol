# Reduction Phase
In the reduction phase, the *candidate* block produced in the [Attestation](../attestation/) phase is validated by two separate committees of [Provisioners](../README.md#participants).

In particular, this phase is divided into two steps, each of which has a committee voting on the validity of the candidate block. 

# Step overview
In each reduction step, selected committee members cast votes on the candidate block.
Votes can be either `nil` to vote against the block, or the block's `hash` to vote in its favor.
Each vote is propagated through the network and collected by all nodes, which accumulate them until a *quorum* is reached.

When a quorum of votes is reached, a [`StepVotes`](#stepvotes) structure is produced, containing all collected votes aggregated, along with the bitset of voters, with respect to the [Voting Committee](../sortition/README.md#voting-committees) of the step.

At the end of the second reduction step, if a quorum is reached, an `Agreement` message is produced with a `StepVotes` structure for each step. This message is passed to the Agreement process, which is responsible for its broadcasting.

If any of the two steps produce a `nil` vote, the Reduction phase will fail and a new iteration will start (i.e., a new Attestation phase will commence).

# Data Structures

#### Reduction Message
ReductionMsg struct {
  hdr        Header
  SignedHash []byte
}

| Field      | Type             | Size      | Description          |
|------------|------------------|-----------|----------------------|
| hdr        | ConsensusHeader  |           | Consensus header     |
| SignedHash | byte[]           | 32 bits   | Signed header     |


#### StepVotes

| Field     | Type   | Size      | Description          |
|-----------|--------|-----------|----------------------|
| BitSet    | uint   | 64 bits   | Bitset of the voters |
| Signature | uint   | 32 bits   | Aggregated signature |

<!-- TODO: explain BitSet -->


# Quorum Handler
When the [Validation][val] and [Ratification][rat] steps reach a quorum, a [Quorum][qmsg] message is produced, which contains a [Certificate][cert] with the aggregated votes of the two quorum committees.
Since the certificate proves a candidate reached a quorum, receiving this message is sufficient to accept the candidate into the local chain.

The Quorum handler task manages [Quorum][qmsg] messages produced by the node or received from the network. In particular, when such a message is received (or produced), the corresponding candidate block is accepted.

### ToC
  - [Overview](#overview)
  - [HandleQuorum](#handlequorum)
    - [VerifyQuorum](#VerifyQuorum)

## Overview
When a node generates or receives a $\mathsf{Quorum}$ message, it adds the included $\mathsf{Certificate}$ to the corresponding candidate block and makes it a *winning block* for the corresponding round and iteration.

If the $\mathsf{Quorum}$ message was received from the network, the aggregated votes (see [`StepVotes`][sv]) are verified against the generator and committee members of the corresponding round and iteration. Both the validity of the votes and the quorum quota are verified.

The winning block will then be accepted as the new tip of the local chain.


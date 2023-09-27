# Reduction Phase
In the Reduction phase, the *candidate* block produced in the [Attestation][att] phase is validated by a subset of [provisioner][p], who verify the block and cast a vote in favor or against it. The phase is divided into two steps, each of which has a committee (using [*Deterministic Sortition*][ds]) vote on the validity of the block.
If the votes of both committees reach a quorum, an $\mathsf{Agreement}$ message broadcasted, which contains the (quorum) votes of both committees.

### ToC
- [Overview](#overview)
  - [`Reduction` Message](#reduction-message)
  - [`StepVotes`](#stepvotes)
- [Reduction Algorithm](#reduction-algorithm)

## Overview
In each reduction step, selected committee members cast votes on the candidate block, if any. A vote can be either the candidate's hash, to vote in favor, or $NIL$ to vote against. Votes are propagated through the network via $\mathsf{Reduction}$ messages, and collected by other nodes, which accumulate them until a *quorum* is reached.

When a quorum of votes is reached, a $StepVotes$ structure is produced, containing all (and only[^1]) the quorum votes aggregated, along with a bitset of the voters with respect to the [Voting Committee][vc].

If a quorum is reached in both steps, an $\mathsf{Agreement}$ message is produced with the $StepVotes$ structures of both steps. This message is passed to the Agreement process, which is responsible for its propagation in the network.

If any of the two steps produces a $NIL$ result, the Reduction phase will fail and a new iteration will start (i.e., a new Attestation phase will commence).
<!-- Currently, if the first step produces $NIL$, nodes still execute the second step.
This behavior should be avoided. If the goal is to spend time, just wait timeout. -->


### Reduction Message
The $\mathsf{Reduction}$ message is used by a member of a Reduction committee to cast a vote on a candidate block. The vote is expressed by the $Header$'s $BlockHash$ field: if containing a hash, the vote is in favor of the corresponding block; if containing a $NIL$, the vote is against the candidate block of the $Round$ and $Step$ specified in the $Header$.


The message has the following structure:
| Field       | Type                  | Size      | Description           |
|-------------|-----------------------|-----------|-----------------------|
| $Header$    | [*MessageHeader*][mh] | 137 bytes | Message header        |
| $Signature$ | BLS Signature         | 48 bytes  | Signature of $Header$ |

The $\mathsf{Reduction}$ message has a total size of 185 bytes.

### StepVotes
The $StepVotes$ structure is produced at the end of a Reduction step and contains a quorum of votes in favor or against a candidate block.

To specify the committee members, whose vote is included in $Votes$, a bitset is used, with each bit corresponding to a committee member: if the bit is set to $1$, the corresponding member's vote is in $Votes$, otherwise it's not.

The structure is defined as follows:

| Field    | Type          | Size     | Description                                |
|----------|---------------|----------|--------------------------------------------|
| $Voters$ | BitSet        | 64 bits  | Bitset of the voters                       |
| $Votes$  | BLS Signature | 48 bytes | Aggregated $\mathsf{Reduction}$ signatures |

Thus, the $StepVotes$ structure has a total size of 56 bytes.

Note that the 64-bit bitset is enough to represent the maximum number of members in a committee (i.e., [*CommitteeCredits*][cp]).


## Reduction Algorithm
<!-- TODO: Add description; Add #RunReduction? -->

***Parameters***
- $Round$: round number
- $Iteration$: iteration number
- $rstep$ : Reduction step number (1 or 2)
- $\mathsf{B}^c$: candidate block

***Algorithm***
1. Extract voting committee for the step
2. Start timeout
3. If part of the committee:
   1. If candidate is empty:
      1. Set vote $v$ to $NIL$
   2. Otherwise:
      1. Verify candidate block
      2. If block is valid, set vote $v$ to block's hash
      3. Otherwise, set $v$ to $NIL$
   3. Broadcast $\mathsf{Reduction}$ message with vote $v$
4. While timeout is not expired:
   1. If a $\mathsf{Reduction}$ message $\mathsf{M}$ is received for round $r$ and step $s$:
      1. If sender is in the committee
      2. and $\mathsf{M}$'s signature is valid
         1. Propagate message
         2. Set vote $v$ to $\mathsf{M}$'s $BlockHash$ (candidate or $NIL$)
         3. Aggregate $v$ to corresponding aggregated signature
         4. Add sender to corresponding voters bitset
         5. If aggregated $v$ votes reached a quorum:
            1. Create $StepVotes$ $\mathsf{V}$ with aggregated $v$
            2. Output $\mathsf{V}$
 5. If timeout expired:
    1. Increase Reduction timeout
    2. Output $NIL$

***Procedure***

$Reduction( Round, Iteration, rstep, \mathsf{B}^c )$:
- $\sigma^{\mathsf{B}^c}$ : aggregate signature for candidate
- $\boldsymbol{bs}^{\mathsf{B}^c}$ : Voters bitset for candidate
- $\sigma^{NIL}$ : aggregate signature for vote NIL
- $\boldsymbol{bs}^{NIL}$ : Voters bitset for NIL
- $r = Round$
- $s = (Iteration-1) \times 3 + 1 + rstep$
1. $C$ = [*DS*][dsa]$(r,s,CommitteeCredits)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } (pk_\mathcal{N} \in C):$
   1. $\texttt{if } (\mathsf{B}^c == NIL):$
      1. $v = NIL$
   2. $\texttt{else}:$
      1. $isValid$ = [*VerifyBlockHeader*][vbh]$(Tip,\mathsf{B}^c)$
      2. $\texttt{if } (isValid = true) : v =$ *Hash*$`_{SHA3}(\mathsf{H}^{\mathsf{B}^c})`$
      3. $\texttt{else}: v = NIL$
   3. $`\mathsf{M}^R = `$ [*Msg*][msg]$(\mathsf{Reduction}, v)$
      | Field       | Value                       | 
      |-------------|-----------------------------|
      | $Header$    | $\mathsf{H}_{\mathsf{M}^R}$ |
      | $Signature$ | $\sigma_{\mathsf{M}^R}$     |

      <!-- Add | $Vote$ | $v$ | -->

   4. [*Broadcast*][mx]$(\mathsf{M}^R)$
4. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Reduction_1}):$
   1. $\texttt{if } (\mathsf{M}^R =$ [*Receive*][mx]$(\mathsf{Reduction},r,s) \ne NIL):$
      1. $\texttt{if } (pk_{\mathsf{M}^R} \in C)$
      2. $\texttt{and }($*VerifySignature*$(\mathsf{M}^R) = true):$
         1. [*Propagate*][mx]$(\mathsf{M}^R)$
         2. $v = \mathsf{H}_{\mathsf{M}^R}.BlockHash$
         3. *AggregateSig*$(\sigma^v, \sigma_{\mathsf{M}^R})$
         4. $m = m_{pk_{\mathsf{M}^R}}$ \
            $\boldsymbol{bs}^{v}[i_m^C] = 1$
         5. $\texttt{if } ($*countSetBits*$(\boldsymbol{bs}^v) \ge Quorum):$
            1. $\mathsf{V} = (\sigma^v, \boldsymbol{bs}^v)$
            2. $\texttt{output } \mathsf{V}$

 5. $\texttt{if } (\tau_{Now} \gt \tau_{Start}+\tau_{Reduction_{rstep}}):$
    1. *IncreaseTimeout*$(\tau_{Reduction_{rstep}})$
    2. $\texttt{output } NIL$

<!----------------------- FOOTNOTES ----------------------->

[^1]: This means that when creating a $StepVotes$ for vote $v$ only related votes are included.

<!------------------------- LINKS ------------------------->

[sv]: #stepvotes
<!-- Consensus -->
[cp]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#consensus-parameters
[mh]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#message-header
[msg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#message-creation
[mx]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#message-exchange
[p]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#provisioners-and-stakes
<!-- Attestation -->
[att]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/attestation/
<!-- Sortition -->
[ds]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md
[dsa]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#deterministic-sortition-ds
[vc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#voting-committees
<!-- Chain Management -->
[vbh]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#verifyblockheader
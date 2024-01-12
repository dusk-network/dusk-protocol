# Validation

In the Validation phase, the *candidate* block produced in the [Proposal][prop] phase is validated by a committee of [provisioners][p], randomly chosen using [*Deterministic Sortition*][ds]. Each extracted member verifies the candidate's header and state transition, and then casts its vote via a [Validation][rmsg] message.

### ToC
- [Overview](#overview)
- [Validation Algorithm](#validation-step)

## Overview
In each Validation step, selected committee members cast votes on the candidate block, if any has been received. A vote can be either the candidate's hash, to vote in favor, or $NIL$ to vote against. Votes are propagated through the network via $\mathsf{Validation}$ messages and collected by other nodes, which accumulate them until a *quorum* is reached or the step timeout expires.

Votes are signatures of either the candidate's hash or the empty hash ($NIL$). When votes for a specific hash reach a quorum, a $\mathsf{StepVotes}$ structure is generated, which contains the aggregated votes[^1] (as BLS signatures) and a bitset indicating the voters with respect to the [Voting Committee][vc].


## Validation Step
<!-- DOING -->
The input for the Validation step is the candidate block output from the [Proposal][prop] step: if no candidate has been received, the provisioner votes $NIL$; if, instead, the candidate has been received, it is verified and, if valid, a vote is casted by signing the $(round, step, candidate_hash)$ triplet.

While the candidate verification is done only by committee members, all provisioners wait for Validation votes from the network, until the step timeout expires.


When the number of $NIL$ votes exceeds $\frac{1}{3}$ of the $CommitteeCredits$ we consider it as a "non-quorum", since it's not possible to reach the $Quorum$ value with the other votes. We represent this non-quorum values with $NilQuorum$ (see [Consensus Parameters][cp]).

***Parameters***
- $Round$: round number
- $Iteration$: iteration number
- $rstep$ : Validation step number (1 or 2)
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
   3. Broadcast $\mathsf{Validation}$ message with vote $v$
4. While timeout is not expired:
   1. If a $\mathsf{Validation}$ message $\mathsf{M}$ is received for round $r$ and step $s$:
      1. If sender is in the committee
      2. and $\mathsf{M}$'s signature is valid
         1. Propagate message
         2. Set vote $v$ to $\mathsf{M}$'s $BlockHash$ (candidate or $NIL$)
         3. Aggregate $v$ to corresponding aggregated signature
         4. Add sender to corresponding voters bitset
         5. If $v$ is $NIL$ and votes reached $NilQuorum$
         6. or $v$ is not $NIL$ and votes reached $Quorum$:
            1. Create $\mathsf{StepVotes}$ $\mathsf{V}$ with aggregated $v$
            2. Output $v$ and $\mathsf{V}$
 5. If timeout expired:
    1. Increase Validation timeout
    2. Output $NIL$ vote and $NIL$ $\mathsf{StepVotes}$

***Procedure***

$Validation( Round, Iteration, rstep, \mathsf{B}^c )$:
- $\sigma^{\mathsf{B}^c}$ : aggregate signature for candidate
- $\boldsymbol{bs}^{\mathsf{B}^c}$ : Voters bitset for candidate
- $\sigma^{NIL}$ : aggregate signature for vote NIL
- $\boldsymbol{bs}^{NIL}$ : Voters bitset for NIL
- $r = Round$
- $s = Iteration \times 3 + rstep$

1. $C$ = [*DS*][dsa]$(r,s,CommitteeCredits)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } (pk_\mathcal{N} \in C):$
   1. $\texttt{if } (\mathsf{B}^c == NIL):$
      1. $v = NIL$
   2. $\texttt{else}:$
      1. $isValid$ = [*VerifyBlockHeader*][vbh]$(Tip,\mathsf{B}^c)$
      2. $\texttt{if } (isValid = true) : v =$ *Hash*$`_{SHA3}(\mathsf{H}^{\mathsf{B}^c})`$
      3. $\texttt{else}: v = NIL$
   3. $`\mathsf{M}^R = `$ [*Msg*][msg]$(\mathsf{Validation}, v)$
      | Field       | Value                       | 
      |-------------|-----------------------------|
      | $Header$    | $\mathsf{H}_{\mathsf{M}^R}$ |
      | $Signature$ | $\sigma_{\mathsf{M}^R}$     |

      <!-- Add | $Vote$ | $v$ | -->

   4. [*Broadcast*][mx]$(\mathsf{M}^R)$
4. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Validation_1}):$
   1. $\texttt{if } (\mathsf{M}^R =$ [*Receive*][mx]$(\mathsf{Validation},r,s) \ne NIL):$
      1. $\texttt{if } (pk_{\mathsf{M}^R} \in C)$
      2. $\texttt{and }($*VerifySignature*$(\mathsf{M}^R) = true):$
         1. [*Propagate*][mx]$(\mathsf{M}^R)$
         2. $v = \mathsf{H}_{\mathsf{M}^R}.BlockHash$
         3. *AggregateSig*$(\sigma^v, \sigma_{\mathsf{M}^R})$
         4. $m = m_{pk_{\mathsf{M}^R}}$ \
            $\boldsymbol{bs}^{v}[i_m^C] = 1$
         5. $\texttt{if } (v=NIL \texttt{ and } $*countSetBits*$(\boldsymbol{bs}^v) \ge NilQuorum)$
         6. $\texttt{or } (v \ne NIL \texttt{ and }$*countSetBits*$(\boldsymbol{bs}^v) \ge Quorum):$
            1. $\mathsf{V} = (\sigma^v, \boldsymbol{bs}^v)$
            2. $\texttt{output } (v, \mathsf{V})$

 5. $\texttt{if } (\tau_{Now} \gt \tau_{Start}+\tau_{Validation_{rstep}}):$
    1. *IncreaseTimeout*$(\tau_{Validation_{rstep}})$
    2. $\texttt{output } (NIL, NIL)$

<!----------------------- FOOTNOTES ----------------------->

[^1]: When creating a $\mathsf{StepVotes}$ for vote $v$, only related votes are included.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md -->

<!-- Consensus -->
[cp]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#consensus-parameters
[p]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#provisioners-and-stakes
<!-- Proposal -->
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal
<!-- Sortition -->
[ds]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md
[dsa]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#deterministic-sortition-ds
[vc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#voting-committees
<!-- Chain Management -->
[vbh]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#verifyblockheader
<!-- Messages -->
[msg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-creation
[mx]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-exchange
[rmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#reduction-message

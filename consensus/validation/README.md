# Validation

In the Validation phase, the *candidate* block produced in the [Proposal][prop] phase is validated by a committee of [provisioners][p], randomly chosen using [*Deterministic Sortition*][ds]. Each extracted member verifies the candidate's header and state transition, and then casts its vote via a [Validation][rmsg] message.

### ToC
- [Overview](#overview)
- [Validation Algorithm](#validation-step)

## Overview
In each Validation step, members of a [Voting Committee][vc] cast votes on the candidate block, if known. A vote can be either the candidate's hash, to vote in favor, or $NIL$ to vote against. Votes are propagated through the network via $\mathsf{Validation}$ messages and collected by other nodes, which accumulate them until a *quorum* is reached or the step timeout expires.

Votes are signatures of either the candidate's hash or the empty hash ($NIL$). When votes for a specific hash reach a quorum, a $\mathsf{StepVotes}$ structure is output, with the aggregated votes of the [quorum committee][sc].


## Validation Step
The Validation step takes as input the candidate block output from the [Proposal][prop] step and executes two actions: 
1. first, if the node's provisioner has been selected in the Validation committee, it votes on the candidate:
   - if a candidate has been received, it is validated against the local $Tip$ and, if valid, a $\text{"Valid"}$ vote is cast by signing the triplet $(round, step, candidate_hash)$; 
   <!-- TODO: if the validation fails, an $Invalid$ vote is cast. vote definition must be changed to support this -->
   - if no candidate has been received during the Proposal step, a $\text{"Timeout"}$ vote is cast, by signing the triplet $(round, step, empty_hash)$.
2. then, the node collects Validation votes from the other members of the committee until a quorum is reached or the step timeout expires:
   - if $\text{"Valid"}$ votes reach the $Quorum$ threshold, the step outputs $\text{"Quorum"}$;
   - if $\text{"Timeout"}$ votes exceeds $\frac{1}{3}$ of the $CommitteeCredits$, it is considered as a "non-quorum" (since it's not possible to reach the $Quorum$ value with the other votes) and a $\text{"NilQuorum"}$ is output;
   - if the timeout expires before receiving enough votes, the step output $\text{"Timeout"}$.

For each vote $v$ ($\text{"Valid"}$/$\text{"Invalid"}$/$\text{"Timeout"}$), a [`StepVotes`][sv] structure $\mathsf{SV}_v=(\sigma_v,\boldsymbol{bs}_v)$ is used to aggregate collected votes. 


***Parameters***
- $R$: round number
- $I$: iteration number
- $\mathsf{B}^c$: candidate block

***Algorithm***
1. Extract voting committee for the step
2. Start step timeout
3. If the node $\mathcal{N}$ is part of the committee:
   1. If candidate $\mathsf{B}^c$ is empty:
      1. Set vote $v$ to $NIL$ ($\text{"Timeout"}$)
   2. Otherwise:
      1. Verify $\mathsf{B}^c$
      2. If $\mathsf{B}^c$ is valid, set vote $v$ to $\mathsf{B}^c$'s hash $\eta_\mathsf{B}^c$
      3. Otherwise, set $v$ to $NIL$ ($\text{"Invalid"}$)
   3. Create $\mathsf{Validation}$ message $\mathsf{M}$ with vote $v$
   4. Broadcast $\mathsf{M}$

4. While timeout has not expired:
   1. If a $\mathsf{Validation}$ message $\mathsf{M}$ is received for round $R$ and iteration $I$:
      1. If $\mathsf{M}$'s signer is in the committee
      2. and the signature is valid
         1. Propagate $\mathsf{M}$
         2. Set vote $v$ to $\mathsf{M}$'s $BlockHash$
         3. Add $\mathsf{M}.Signature$ $\mathsf{SV}_v$
         4. If $v$ is a candidate hash and votes in $\mathsf{SV}_v$ reach $Quorum$
            1. Output $\text{"Quorum"}$ and $\mathsf{SV}_v$
         5. If $v$ is $NIL$ and votes in $\mathsf{SV}_v$ reach $NilQuorum$:
            1. Output $\text{"NilQuorum"}$ and $\mathsf{SV}_v$
 5. If timeout expired:
    1. Increase Validation timeout
    2. Output $\text{"Timeout"}$ vote and $\mathsf{SV}_{NIL}$
    <!-- TODO: why do we output SV_NIL? what if we have all Valid votes but less than Quorum? -->

***Procedure***

$Validation( R, I, \mathsf{B}^c ) :$
- $\texttt{set}:$ 
  - $s = I \times 3 + 1$
1. $\mathsf{C}$ = [*DS*][dsa]$(R,s,CommitteeCredits)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } (pk_\mathcal{N} \in \mathsf{C}):$
   1. $\texttt{if } (\mathsf{B}^c = NIL):$
      1. $v = NIL$
   2. $\texttt{else}:$
      1. $isValid$ = [*VerifyBlockHeader*][vbh]$(Tip,\mathsf{B}^c)$
      2. $\texttt{if } (isValid = true) : v = \eta_{\mathsf{B}^c}$
      3. $\texttt{else}: v = NIL$
   3. $`\mathsf{M} = `$ [*Msg*][msg]$(\mathsf{Validation}, v)$
      | Field       | Value                     | 
      |-------------|---------------------------|
      | $Header$    | $\mathsf{H}_{\mathsf{M}}$ |
      | $Signature$ | $\sigma_{\mathsf{M}}$     |

   4. [*Broadcast*][mx]$(\mathsf{M})$

- $\texttt{set}:$
   - $\mathsf{SV}_c = (\sigma_{\mathsf{B}^c}, \boldsymbol{bs}_{\mathsf{B}^c})$
   - $\mathsf{SV}_{NIL} = (\sigma_{NIL}, \boldsymbol{bs}_{NIL})$

4. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Validation}):$
   1. $\texttt{if } (\mathsf{M} =$ [*Receive*][mx]$(\mathsf{Validation},R,I) \ne NIL):$
      1. $\texttt{if } (pk_{\mathsf{M}} \in \mathsf{C})$
      2. $\texttt{and }($*VerifySignature*$(\mathsf{M}) = true):$
         1. [*Propagate*][mx]$(\mathsf{M})$
         2. $v = \mathsf{H}_{\mathsf{M}}.BlockHash$
         3. $\mathsf{SV}_v = $[*AggregateVote*][av]$( \mathsf{SV}_v, \mathsf{C}, \sigma_{\mathsf{M}}, pk_{\mathsf{M}} )$
         4. $\texttt{if } (v \ne NIL \texttt{ and }$[*countSetBits*][cb]$(\boldsymbol{bs}_v) \ge Quorum):$
            1. $\texttt{output } (\text{"ValidQuorum"},, \mathsf{SV}_v)$
         5. $\texttt{if } (v=NIL \texttt{ and } $[*countSetBits*][cb]$(\boldsymbol{bs}_v) \ge NilQuorum):$
            1. $\texttt{output } (\text{"NilQuorum"}, \mathsf{SV}_v)$

 5. $\texttt{if } (\tau_{Now} \gt \tau_{Start}+\tau_{Validation}):$
    1. [*IncreaseTimeout*][it]$(\tau_{Validation})$
    2. $\texttt{output } (\text{"Timeout"}, \mathsf{SV}_{NIL})$


<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md -->
[val]: #validation-step


<!-- Consensus -->
[cp]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#consensus-parameters
[p]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#provisioners-and-stakes
[sv]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#stepvotes
[av]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#aggregatevote
[it]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#increasetimeout
<!-- Proposal -->
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal
<!-- Sortition -->
[ds]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md
[dsa]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#deterministic-sortition-ds
[vc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#voting-committees
[sc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#subcommittees
[cb]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#countsetbits
<!-- Chain Management -->
[vbh]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#verifyblockheader
<!-- Messages -->
[msg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-creation
[mx]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-exchange
[rmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#reduction-message

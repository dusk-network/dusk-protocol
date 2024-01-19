# Validation
*Validation* is the second step in an [*SA iteration*][sai]. In this step, the *candidate* block, produced or received in the [Proposal][prop] step, is validated by a committee of randomly chosen provisioners.

Members of the extracted committee verify the candidate's validity and then cast their vote accordingly. At the same time, all provisioners, including committee members, collect Validation votes from the network.

### ToC
  - [Overview](#overview)
  - [Validation Step](#validation-step)
    - [Procedures](#procedures)
      - [*ValidationStep*](#validationstep)


## Overview
In the Validation step, each node first executes the [*Deterministic Sortition*][ds] algorithm to extract the [Voting Committee][vc] for the step.

If the node is part of the committee, it validates the output from the [Proposal][prop] step. If the output was $NIL$, it votes $NoCandidate$. Otherwise, it verifies the candidate block's validity with respect to the previous block (i.e., the node's local $Tip$). If the candidate is valid, the node votes $Valid$, otherwise, it votes $Invalid$.
The vote is broadcast using a $\mathsf{Validation}$ message (see [`Validation`][vmsg]).

In the same step, all nodes, including the committee members, collect votes from the network until a *quorum* of $\frac{2}{3}$ of the committee credits[^1] is reached, a *negative quorum* ($\frac{1}{3}{+}1$) of non-valid votes is reached[^2], or the step timeout expires. 

If a quorum of $Valid$ votes is received, the step outputs $ValidQuorum$. If a quorum of $Invalid$ votes is received, the step outputs $InvalidQuorum$. If the step timeout expires and a negative quorum of $NoCandidate$ and $Invalid$ votes are received, the step outputs $Fail$, which represents the negative result where no $ValidQuorum$ can be reached. 
If collected votes are insufficient, the step outputs $NoQuorum$, which represents an unknown result (i.e., it is possible that casted votes reach a quorum or a negative quorum but the node has not seen it).

In all cases except $NoQuorum$, the output also includes the aggregated votes of the quorum, or negative quorum, in a $\mathsf{StepVotes}$ structure, which contains the aggregated signatures of the votes and a bitset to identify the voters.

The step output will then be passed on as input to the [Ratification][rat] step.

### Procedures

#### *ValidationStep*
*ValidationStep* takes in input the round $R$, the iteration $I$, and the candidate block $\mathsf{B}^c$ output by [*ProposalStep*][ps] and outputs the result of the collected Validation votes ($ValidQuorum$, $InvalidQuorum$, $Fail$, $NoQuorum$) plus the aggregated votes $\mathsf{SV}$ that produced the result.

In the procedure, the node performs two main tasks: 

1. if selected in the Validation committee $\mathsf{C}$, it checks the candidate $\mathsf{B}^c$ and broadcasts a $\mathsf{Validation}$ message with its vote: $Valid$ if $\mathsf{B}^c$ is a valid successor of the local $Tip$, $Invalid$ if it's not, and $NoCandidate$ if it's $NIL$ (no candidate has been received).

1. it collects $\mathsf{Validation}$ messages from all committee members, and sets the result depending on the votes:
   - if $Valid$ votes reach $Quorum$, the step outputs $ValidQuorum$;
   - if $Invalid$ votes reach $Quorum$, the step outputs $InvalidQuorum$;
   - if $NoCandidate$ and $Invalid$ votes combined reach $NegativeQuorum$, the step outputs $Fail$;
   - if the timeout $\tau_{Validation}$ expires, the step outputs $NoQuorum$.

Collected votes are aggregated in [`StepVotes`][sv] structures. In particular, for each vote $v$ ($Valid$ / $Invalid$ / $NoCandidate$ / $NoQuorum$), a $\mathsf{SV}_v=(\sigma_v,\boldsymbol{bs}_v)$ is used.

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

$ValidationStep( R, I, \mathsf{B}^c ) :$
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
    2. $\texttt{output } (\text{"NoQuorum"}, \mathsf{SV}_{NIL})$


<!----------------------- FOOTNOTES ----------------------->
[^1]: remember that the quorum is calculated over the weight of the voters, not their number.
[^2]: We use the term *negative-quorum* to indicate the threshold of $\frac{1}{3}+1$ of non-valid votes, which mathematically determines the impossibility of a *quorum* ($\frac{2}{3}$) to be reached. Negative quorums are useful for *attesting* failed iterations (see [rolling finality][rf]).

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md -->
[val]: #validation-step


<!-- Consensus -->
[cp]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#consensus-parameters
[p]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#provisioners-and-stakes
[sv]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#stepvotes
[av]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#aggregatevote
[it]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#increasetimeout
[sai]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#saiteration
<!-- Proposal -->
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal
[ps]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal#proposalstep
<!-- Ratification -->
[rat]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification
<!-- Sortition -->
[ds]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md
[dsa]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#deterministic-sortition-ds
[vc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#voting-committees
[sc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#subcommittees
[cb]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#countsetbits
<!-- Chain Management -->
[vbh]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#verifyblockheader
[rf]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#rolling-finality
<!-- Messages -->
[msg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-creation
[mx]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-exchange
[vmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#validation-message

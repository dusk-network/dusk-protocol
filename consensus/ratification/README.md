# Ratification
*Ratification* is the third step in an [*SA iteration*][sai]. In this step, the result of the [Validation][val] step is agreed upon by another committee of randomly chosen provisioners.

Members of the extracted committee cast a vote with the output of the[Validation][step] as resulting from the $\mathsf{Validation}$ messages received. At the same time, all provisioners, including committee members, collect Ratification votes from the network until a target quorum is reached or the step timeout expires.

If a quorum is reached for any result, a $mathsf{Quorum}$ [`Quorum`][qmsg] is generated with the aggregated signatures of both Validation and Ratification steps.

The main purpose of the Ratification step is to ensure provisioners are "aligned" with respect to the Validation result: if Validation result was $Valid$, it ensures a supermajority of provisioners saw such a result and hence accepted the block. Similarly, in case of non-$Valid$ result, it ensures a majority of provisioners will certify this iteration as failed, which, in turn, is used in determining if a winning candidate will be Attested or not (see [Finality][fin]).

### ToC
- [Overview](#overview)
- [Ratification Step](#ratification-step)
  - [Procedures](#procedures)
    - [*RatificationStep*](#ratificationstep)


## Overview
In the Ratification step, each node first executes the [*Deterministic Sortition*][ds] algorithm to extract the [Voting Committee][vc] for the step.

If the node is part of the committee, it casts a vote with the winning [Validation][val] vote ($Valid$, $Invalid$, $NoCandidate$, $NoQuorum$). 
The vote is broadcast using a $\mathsf{Ratification}$ message (see [`Ratification`][rmsg]), which includes the $\mathsf{StepVotes}$ of the aggregated Validation votes.

Then, all nodes, including the committee members, collect votes from the network until a *supermajority*  ($\frac{2}{3}$ of the committee credits) of $Valid$ votes is reached, a *majority* ($\frac{1}{2}{+}1$) of non-$Valid$ votes is reached, or the step timeout expires. 
Ratification votes (except for $NoQuorum$) are only considered valid if accompanied by a $\mathsf{StepVotes}$ with a quorum of matching Validation votes.

Note that, since Ratification votes require a quorum of Validation votes (except for $NoQuorum$) to be valid, only one vote between $Valid$, $Invalid$, and $NoCandidate$ can be received, plus $NoQuorum$. In other words, $Valid$, $Invalid$, and $NoCandidate$ are mutually exclusive in Ratification. In fact, it's impossible to receive a valid $NoCandidate$ vote (which requires a majority) and $Valid$ vote in the same Ratification step, unless a provisioner casts a double vote[^1].

If any quorum is reached, the step outputs the winning vote ($Valid$, $Invalid$, $NoCandidate$). Otherwise, if the step timeout expires, the step outputs $NoQuorum$.
In all cases, except $NoQuorum$, the output of the step includes a $\mathsf{StepVotes}$ structure (see [`StepVotes`][sv]) with the aggregated votes that determined the result.

The output, together with the Validation output, will be used to determine the outcome of the iteration.


### Procedures
<!-- DOING -->


#### *RatificationStep*
*RatificationStep* takes in input the round $R$, the iteration $I$, and the candidate block $\mathsf{B}^c$ (as returned by [*ProposalStep*][ps]) and outputs the result of the collected Validation votes ($Valid$, $Invalid$, $NoCandidate$, or $NoQuorum$) plus the aggregated votes $\mathsf{SV}$ that produced the result.

The procedure performs two main tasks: 

1. if the node is selected in the Validation committee $\mathsf{C}$, it verifies the candidate $\mathsf{B}^c$ and broadcasts a $\mathsf{Validation}$ message with its vote: $Valid$ if $\mathsf{B}^c$ is a valid successor of the local $Tip$, $Invalid$ if it's not, and $NoCandidate$ if it's $NIL$ (no candidate has been received).

2. it then collects $\mathsf{Validation}$ messages from all committee members, and sets the result depending on the votes:
   - if $Valid$ votes reach $Quorum$, the step outputs $Valid$;
   - if $Invalid$ votes reach $Majority$, the step outputs $Invalid$;
   - if $NoCandidate$ votes reach $Majority$, the step outputs $NoCandidate$;
   - if the timeout $\tau_{Validation}$ expires, the step outputs $NoQuorum$.

Collected votes are aggregated in [`StepVotes`][sv] structures. In particular, for each vote $v$ ($Valid$ / $Invalid$ / $NoCandidate$ / $NoQuorum$), a $\mathsf{SV}_v=(\sigma_v,\boldsymbol{bs}_v)$ is used.

***Parameters***
- $R$: round number
- $I$: iteration number
- $\mathsf{B}^c$: candidate block

***Algorithm***
1. Extract committee $\mathsf{C}$ for the step
2. Start step timeout $\tau_{Validation}$
3. If the node $\mathcal{N}$ is part of $\mathsf{C}$:
   1. If candidate $\mathsf{B}^c$ is empty:
      1. Set vote $v$ to $NoCandidate$
   2. Otherwise:
      1. Verify $\mathsf{B}^c$
      2. If $\mathsf{B}^c$ is valid, set vote $v$ to $Valid$
      3. Otherwise, set $v$ to $Invalid$
   3. Create $\mathsf{Validation}$ message $\mathsf{M}$ for vote $v$
   4. Broadcast $\mathsf{M}$

4. For each vote $v$ ($Valid$, $Invalid$, $NoCandidate$)
   1. Initialize $\mathsf{SV}_v$

5. While timeout $$\tau_{Validation}$ has not expired:
   1. If a $\mathsf{Validation}$ message $\mathsf{M}$ is received for round $R$ and iteration $I$:
      1. If $\mathsf{M}$'s signature is valid
      2. and $\mathsf{M}$'s signer is in the committee $\mathsf{C}$
         1. Propagate $\mathsf{M}$
         2. Collect $\mathsf{M}$'s vote $v$ into the aggregated $\mathsf{SV}_v$
         3. Set target $Target$ to $Quorum$ if $v$ is $Valid$ or $Majority$ if $v$ is $Invalid$ or $NoCandidate$
         4. If votes in $\mathsf{SV}_v$ reach $Target$
            1. Output $(v, \mathsf{SV}_v)$

 6. If timeout $\tau_{Validation}$ expired:
    1. Increase Validation timeout
    2. Output $(NoQuorum, NIL)$

***Procedure***

$RatificationStep( R, I, \mathsf{B}^c ) :$
- $\texttt{set}:$ 
  - $s = I \times 3 + 1$
1. $\mathsf{C}$ = [*DS*][dsa]$(R,s,CommitteeCredits)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } (pk_\mathcal{N} \in \mathsf{C}):$
   1. $\texttt{if } (\mathsf{B}^c = NIL):$
      1. $v = NoCandidate$
   2. $\texttt{else}:$
      1. $isValid$ = [*VerifyBlockHeader*][vbh]$(Tip,\mathsf{B}^c)$
      2. $\texttt{if } (isValid = true) : v = Valid$
      3. $\texttt{else}: v = Invalid$
   3. $`\mathsf{M} = `$ [*Msg*][msg]$(\mathsf{Validation}, v)$
      <!-- TODO: update when updating Consensus Message definition
      | Field       | Value                     | 
      |-------------|---------------------------|
      | $Header$    | $\mathsf{H}_{\mathsf{M}}$ |
      | $Signature$ | $\sigma_{\mathsf{M}}$     | 
      -->

   4. [*Broadcast*][mx]$(\mathsf{M})$

4. $\texttt{set}:$
   - $\texttt{for } v \texttt{ in } [Valid, Invalid, NoCandidate]:$
     - $\mathsf{SV}_v = (\sigma_v, \boldsymbol{bs}_v)$

5. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Validation}):$
   1. $\texttt{if } (\mathsf{M} =$ [*Receive*][mx]$(\mathsf{Validation},R,I) \ne NIL):$
      1. $\texttt{if } (pk_{\mathsf{M}} \in \mathsf{C})$
      2. $\texttt{and }($*VerifySignature*$(\mathsf{M}) = true):$
         1. [*Propagate*][mx]$(\mathsf{M})$
         2. $v = \mathsf{M}.Vote$
         3. $\mathsf{SV}_v = $[*AggregateVote*][av]$( \mathsf{SV}_v, \mathsf{C}, \mathsf{M}.Signature, pk_{\mathsf{M}} )$
         4. $\texttt{set}:$
            - $\texttt{if } (v = Valid): Target = Quorum$
            - $\texttt{else}: Target = Majority$
         5. $\texttt{if }($[*countSetBits*][cb]$(\boldsymbol{bs}_v) \ge Target):$
            1. $\texttt{output } (v, \mathsf{SV}_v)$

 6. $\texttt{if } (\tau_{Now} \gt \tau_{Start}+\tau_{Validation}):$
    1. [*IncreaseTimeout*][it]$(\tau_{Validation})$
    2. $\texttt{output } (NoQuorum, NIL)$


<!----------------------- FOOTNOTES ----------------------->
[^1]: We currently assume no double-votes are casted. In a future version of the protocol, double votes will be punished with slashing.

<!-- [^2]: We use the term *negative-quorum* to indicate the threshold of $\frac{1}{3}+1$ of non-valid votes, which mathematically determines the impossibility of a *quorum* ($\frac{2}{3}$) to be reached. Negative quorums are useful for *attesting* failed iterations (see [rolling finality][rf]). -->

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md -->
[rat]: #ratification-step


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
<!-- Validation -->
[rat]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation
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

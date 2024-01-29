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

#### *RatificationStep*
*RatificationStep* takes in input the round $R$, the iteration $I$, and the Validation result $\mathsf{SR}^V$ (as returned from [*ValidationStep*][vals]) and outputs the Ratification result $\mathsf{SR}^R=(v^R, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^R})$, where $v^R$ is $Valid$, $Invalid$, $NoCandidate$, or $NoQuorum$, $\eta_{\mathsf{B}^c}$ is the candidate hash, and $\mathsf{SV}_{v^R}$ is the aggregated vote of the quorum committee.

The procedure performs two tasks: 

1. If the node is part of the Ratification committee $\mathsf{C}$, it broadcasts a $\mathsf{Ratification}$ message with the Validation result $(v^V, \mathsf{SV}_{v^V})$.

2. It collects $\mathsf{Ratification}$ messages from all committee members, and sets the result depending on the votes:
   - if $Valid$ votes reach $Supermajority$, the step outputs $Valid$;
   - if $Invalid$ votes reach $Majority$, the step outputs $Invalid$;
   - if $NoCandidate$ votes reach $Majority$, the step outputs $NoCandidate$;
   - if the timeout $\tau_{Ratification}$ expires, the step outputs $NoQuorum$.

Collected votes are aggregated in [`StepVotes`][sv] structures. In particular, for each vote $v$ ($Valid$ / $Invalid$ / $NoCandidate$ / $NoQuorum$), a $\mathsf{SV}_v=(\sigma_v,\boldsymbol{bs}_v)$ is used.

***Parameters***
- $R$: round number
- $I$: iteration number
- $\mathsf{SR}^V = ($v^V, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^V})$ Validation result ([`StepResult`][sr])

***Algorithm***
1. Extract committee $\mathsf{C}$ for the step
2. Start step timeout $\tau_{Ratification}$
3. If the node $\mathcal{N}$ is part of $\mathsf{C}$:
   1. Create a $\mathsf{Ratification}$ message $\mathsf{M}$ for vote $v^V$
   2. Broadcast $\mathsf{M}$

4. For each vote $v$ ($Valid$, $Invalid$, $NoCandidate$, $NoQuorum$)
   1. Initialize $\mathsf{SV}_v$

5. While timeout $\tau_{Ratification}$ has not expired:
   1. If a $\mathsf{Ratification}$ message $\mathsf{M}$ is received for round $R$ and iteration $I$:
      1. If $\mathsf{M}$'s signature is valid
      2. and $\mathsf{M}$'s signer is in the committee $\mathsf{C}$
         1. Propagate $\mathsf{M}$
         2. Collect $\mathsf{M}$'s vote $v^R$ into the aggregated $\mathsf{SV}_{v^R}$
         3. Set the target quorum $Q$ to $Supermajority$ if $v^R$ is $Valid$ or $Majority$ if $v^R$ is $Invalid$, $NoCandidate$, or $NoQuorum$
         4. If votes in $\mathsf{SV}_{v^R}$ reach $Q$
            1. Output $(v^R, \mathsf{SV}_{v^R})$

 6. If timeout $\tau_{Ratification}$ expired:
    1. Increase Ratification timeout
    2. Output $(NoQuorum, NIL)$

***Procedure***

$RatificationStep( R, I, \mathsf{SR}^V ) :$
- $\texttt{set}:$ 
  - $s = I \times 3 + 1$
  - $v^V, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^V} \leftarrow \mathsf{SR}^V$
1. $\mathsf{C}$ = [*DS*][dsa]$(R,S,CommitteeCredits)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } (pk_\mathcal{N} \in \mathsf{C}):$
   1. $`\mathsf{M} = `$ [*Msg*][msg]$(\mathsf{Ratification}, v^V, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^V})$
      | Field             | Value                     | 
      |-------------------|---------------------------|
      | $Header$          | $\mathsf{H}_{\mathsf{M}}$ |
      | $Vote$            | $v^V$                     | 
      | $CandidateHash$   | $\eta_{\mathsf{B}^c}$     |
      | $ValidationVotes$ | $\mathsf{SV}_{v^V}$       |

   2. [*Broadcast*][mx]$(\mathsf{M})$

4. $\texttt{set}:$
   - $\texttt{for } v \texttt{ in } [Valid, Invalid, NoCandidate, NoQuorum]:$
     - $\mathsf{SV}_v = (\sigma_v, \boldsymbol{bs}_v)$

5. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Ratification}):$
   1. $\texttt{if } (\mathsf{M} =$ [*Receive*][mx]$(\mathsf{Ratification},R,I) \ne NIL):$
      - $\texttt{set: } \sigma_\mathsf{M} = \mathsf{M}.Signature$
      1. $\texttt{if } (pk_\mathsf{M} \in \mathsf{C})$
      2. $\texttt{and }($*VerifySignature*$(\sigma_\mathsf{M}, pk_\mathsf{M}) = true):$
         1. [*Propagate*][mx]$(\mathsf{M})$
         2. $v^R = \mathsf{M}.Vote$
         3. $\mathsf{SV}_{v^R} =$ [*AggregateVote*][av]$( \mathsf{SV}_{v^R}, \mathsf{C}, \sigma_\mathsf{M}, pk_{\mathsf{M}} )$
         4. $\texttt{set}:$
            - $\texttt{if } (v^R = Valid): Q = Supermajority$
            - $\texttt{else}: Q = Majority$
         5. $\texttt{if }($[*countSetBits*][cb]$(\boldsymbol{bs}_{v^R}) \ge Q):$
            1. $\texttt{output } (v^R, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^R})$

 6. $\texttt{if } (\tau_{Now} \gt \tau_{Start}+\tau_{Ratification}):$
    1. [*IncreaseTimeout*][it]$(\tau_{Ratification})$
    2. $\texttt{output } (NoQuorum, NIL, NIL)$


<!----------------------- FOOTNOTES ----------------------->
[^1]: We currently assume no double votes are cast. In a future version of the protocol, double votes will be punished with slashing.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md -->
[rs]: #ratificationstep

<!-- Basics -->
[sr]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#stepresult
[sv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#stepvotes

<!-- Consensus -->
[env]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#environment
[p]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#provisioners-and-stakes
[av]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#aggregatevote
[it]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#increasetimeout
[sai]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#saiteration

<!-- Proposal -->
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal
[ps]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal#proposalstep

<!-- Validation -->
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md
[vals]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md#validation-step

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
[mx]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#procedures
[vmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#validation-message

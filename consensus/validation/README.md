# Validation
*Validation* is the second step in an [*SA iteration*][sai]. In this step, the *candidate* block, produced or received in the [Proposal][prop] step, is validated by a committee of randomly chosen provisioners.

Members of the extracted committee verify the candidate's validity and then cast their vote accordingly. At the same time, all provisioners, including committee members, collect Validation votes from the network until a target quorum is reached, or the step timeout expires.

The main purpose of the Validation step is to agree on whether a candidate block was produced and if it was valid against its parent block.

### ToC
  - [Overview](#overview)
  - [Validation Step](#validation-step)
    - [Procedures](#procedures)
      - [*ValidationStep*](#validationstep)


## Overview
In the Validation step, each node first executes the [*Deterministic Sortition*][ds] algorithm to extract the [Voting Committee][vc] for the step.

If the node is part of the committee, it validates the output from the [Proposal][prop] step. If the output was $NIL$, it votes $NoCandidate$. Otherwise, it verifies the candidate block's validity against its previous block (i.e., the node's local $Tip$). If the candidate is valid, the node votes $Valid$, otherwise, it votes $Invalid$.
Non-$Valid$ outputs are used to prove an iteration failed (i.e., it can't reach a quorum of $Valid$ votes), which is functional to block [*Attestation*][fin]; additionally, these votes are used for [slashing][sla].
The vote is broadcast using a $\mathsf{Validation}$ message (see [`Validation`][vmsg]).

Then, all nodes, including the committee members, collect votes from the network until a *supermajority* ($\frac{2}{3}$ of the committee credits[^1]) of $Valid$ votes is reached, a *majority* ($\frac{1}{2}{+}1$) of non-$Valid$ votes is reached, or the step timeout expires.
Specifically, if a supermajority of $Valid$ votes is received, the step outputs $Valid$; if a majority of $Invalid$ or $NoCandidate$ votes is received, the step outputs $Invalid$ or $NoCandidate$, respectively.
Note that, while $\frac{1}{3}{+}1$ of $Invalid$/$NoCandidate$ votes would be sufficient to prove a failed iteration, waiting for a majority of votes allows reducing the risk of slashing unfairly (e.g., slashing for a missed block based on a minority of votes).

If the step timeout expires, the step outputs $NoQuorum$, which represents an unknown result: it is possible that casted votes reach a quorum or a majority but the node did not see it.

In all cases, except $NoQuorum$, the output of the step includes a $\mathsf{StepVotes}$ structure (see [`StepVotes`][sv]) with the aggregated votes that determined the result.

The step output will be used as the input for the [Ratification][rat] step.

### Procedures

#### *ValidationStep*
*ValidationStep* takes in input the round $R$, the iteration $I$, and the candidate block $\mathsf{B}^c$ (as returned by [*ProposalStep*][ps]) and outputs the Validation result $(v^V, \mathsf{SV}_{v^V})$, where $v^V$ is $Valid$, $Invalid$, $NoCandidate$, or $NoQuorum$, and $\mathsf{SV}_{v^V}$ is the aggregated vote of the quorum committee.

The procedure performs two tasks: 

1. If the node is part of the Validation committee $\mathsf{C}$, it verifies the candidate $\mathsf{B}^c$ and broadcasts a $\mathsf{Validation}$ message with its vote: $Valid$ if $\mathsf{B}^c$ is a valid successor of the local $Tip$, $Invalid$ if it's not, and $NoCandidate$ if it's $NIL$ (no candidate has been received).

2. It collects $\mathsf{Validation}$ messages from all committee members, and sets the result depending on the votes:
   - if $Valid$ votes reach $Supermajority$, the step outputs $Valid$;
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
      1. Verify $\mathsf{B}^c$ against $Tip$
      2. If $\mathsf{B}^c$ is valid, set vote $v$ to $Valid$
      3. Otherwise, set $v$ to $Invalid$
   3. Set $VoteInfo$ to $(v, \eta_{\mathsf{B}^c})$
   4. Create a $\mathsf{Validation}$ message $\mathsf{M}$ for vote $v$
   5. Broadcast $\mathsf{M}$

4. For each vote $v$ ($Valid$, $Invalid$, $NoCandidate$)
   1. Initialize $\mathsf{SV}_v$

5. While timeout $\tau_{Validation}$ has not expired:
   1. If a $\mathsf{Validation}$ message $\mathsf{M^V}$ is received for round $R$ and iteration $I$:
      1. If $\mathsf{M^V}$'s signature is valid
      2. and $\mathsf{M^V}$'s signer is in the committee $\mathsf{C}$
         1. Propagate $\mathsf{M^V}$
         2. Collect $\mathsf{M^V}$'s vote $v^V$ into the aggregated $\mathsf{SV}_{v^V}$
         3. Set the target quorum $Q$ to $Supermajority$ if $v^V$ is $Valid$, or to $Majority$ if $^V$ is $Invalid$ or $NoCandidate$
         4. If votes in $\mathsf{SV}_{v^V}$ reach $Q$
            1. Output $(v^V, \mathsf{SV}_{v^V})$

 6. If timeout $\tau_{Validation}$ expired:
    1. Increase Validation timeout
    2. Output $(NoQuorum, NIL)$

***Procedure***

$ValidationStep( R, I, \mathsf{B}^c ) :$
- $\texttt{set}:$ 
  - $S =$ [*GetStepNum*][gsn]$(I, ValStep)$
1. $\mathsf{C}$ = [*DS*][dsa]$(R,S,CommitteeCredits)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } (pk_\mathcal{N} \in \mathsf{C}):$
   1. $\texttt{if } (\mathsf{B}^c = NIL):$
      1. $v = NoCandidate$
   2. $\texttt{else}:$
      1. $isValid$ = [*VerifyBlockHeader*][vbh]$(Tip,\mathsf{B}^c)$
      2. $\texttt{if } (isValid = true) : v = Valid$
      3. $\texttt{else}: v = Invalid$
   3. $\texttt{set}: \mathsf{V} = (v, \eta_{\mathsf{B}^c})$
   4. $`\mathsf{M} = `$ [*Msg*][msg]$(\mathsf{Validation}, \mathsf{V})$
      | Field           | Value                 | 
      |-----------------|-----------------------|
      | $PrevHash$      | $\eta_{Tip}$          |
      | $Round$         | $R$                   |
      | $Iteration$     | $I$                   |
      | $Vote$          | $v$                   |
      | $CandidateHash$ | $\eta_{\mathsf{B}^c}$ |
      | $Signer$        | $pk_\mathcal{N}$      |
      | $Signature$     | $\sigma_\mathsf{M}$   |

   5. [*Broadcast*][mx]$(\mathsf{M})$

4. $\texttt{set}:$
   - $\texttt{for } v \texttt{ in } [Valid, Invalid, NoCandidate]:$
     - $\mathsf{SV}_v = (\sigma_v, \boldsymbol{bs}_v)$

5. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Validation}):$
   1. $\texttt{if } (\mathsf{M^V} =$ [*Receive*][mx]$(\mathsf{Validation},R,I) \ne NIL):$
      - $\mathsf{CI}, \mathsf{VI}, \mathsf{SI} \leftarrow \mathsf{M^V}$
      - $\eta_{\mathsf{B}^p}, \_, \_, \leftarrow \mathsf{CI}$
      - $v^V, \eta_{\mathsf{B}^c} \leftarrow \mathsf{VI}$
      - $pk_\mathsf{M^V}, \sigma_\mathsf{M^V} \leftarrow \mathsf{SI}$

      1. $\texttt{if } (pk_{\mathsf{M^V}} \in \mathsf{C})$
      2. $\texttt{and }($[*VerifyMessage*][ms]$(\mathsf{M^V}) = true):$
         1. [*Propagate*][mx]$(\mathsf{M^V})$
         2. $\mathsf{SV}_{v^V} =$ [*AggregateVote*][av]$( \mathsf{SV}_{v^V}, \mathsf{C}, \sigma_\mathsf{M^V}, pk_\mathsf{M^V} )$
         3. $Q =$[*GetQuorum*][gq]$(v^V)$
         4. $\texttt{if }($[*countSetBits*][cb]$(\boldsymbol{bs}_{v^V}) \ge Q):$
            1. $\texttt{output } (v^V, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^V})$

 6. $\texttt{if } (\tau_{Now} \gt \tau_{Start}+\tau_{Validation}):$
    1. [*IncreaseTimeout*][it]$(\tau_{Validation})$
    2. $\texttt{output } (NoQuorum, NIL, NIL)$


<!----------------------- FOOTNOTES ----------------------->
[^1]: remember that the quorum is calculated over the weight of the voters, not their number.

<!------------------------- LINKS ------------------------->
[val]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md
[vals]: #validation-step


<!-- Consensus -->
[env]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#environment
[p]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#provisioners-and-stakes
[sv]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#stepvotes
[av]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#aggregatevote
[it]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#increasetimeout
[sai]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#saiteration
[gsn]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#GetStepNum

[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal
[ps]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal#proposalstep

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
[fin]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#finality

<!-- Messages -->
[ms]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#signatures
[mx]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#procedures
[vmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#validation-message
[msg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-creation

<!-- TODO -->
[sla]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/slashing
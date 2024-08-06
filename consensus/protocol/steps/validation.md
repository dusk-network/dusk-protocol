# Validation
*Validation* is the second step in an [*SA iteration*][sai]. In this step, the *candidate* block, produced or received in the [Proposal][prop] step, is validated by a committee of randomly chosen provisioners.

Members of the extracted committee verify the candidate's validity and then cast their vote accordingly. At the same time, all provisioners, including committee members, collect Validation votes from the network until a target quorum is reached, or the step timeout expires.

The main purpose of the Validation step is to agree on whether a candidate block was produced and if it was valid against its parent block.

**ToC**
  - [Overview](#overview)
    - [Procedures](#procedures)
      - [*ValidationStep*](#validationstep)

## Overview
In the Validation step, each node first executes the [*Deterministic Sortition*][ds] algorithm to extract the [Voting Committee][vc] for the step.

If the node is part of the committee, it validates the output from the [Proposal][prop] step. If the output was $NIL$, it votes $NoCandidate$. Otherwise, it verifies the candidate block's validity against its previous block (i.e., the node's local $Tip$). If the candidate is valid, the node votes $Valid$, otherwise, it votes $Invalid$.
$\text{Non-}Valid$ outputs are used to prove an iteration failed (i.e., it can't reach a quorum of $Valid$ votes), which is functional to block [*finalization*][rf]; additionally, these votes are used to trigger [penalties][pen].
The vote is broadcast using a [`Validation`][vmsg] message.

Then, all nodes, including the committee members, collect votes from the network until a *supermajority* ($\frac{2}{3}$ of the committee credits[^1]) of $Valid$ votes is reached, a *majority* ($\frac{1}{2}{+}1$) of $\text{non-}Valid$ votes is reached, or the step timeout expires.
Specifically, if a supermajority of $Valid$ votes is received, the step outputs $Valid$; if a majority of $Invalid$ or $NoCandidate$ votes is received, the step outputs $Invalid$ or $NoCandidate$, respectively.
Note that, while $\frac{1}{3}{+}1$ of $Invalid$ or $NoCandidate$ votes would be sufficient to prove a failed iteration, waiting for a majority of votes allows reducing the risk of [punishing][pen] unfairly (e.g., slashing for a missed block based on a minority of votes).

If the step timeout expires, the step outputs $NoQuorum$, which represents an unknown result: it is possible that casted votes reach a quorum or a majority but the node did not see it.

In all cases, except $NoQuorum$, the output of the step includes a [`StepVotes`][sv] structure with the aggregated votes that determined the result.

The step output will be used as the input for the [Ratification][rat] step.

### Procedures

#### *ValidationStep*
<!-- TODO: use Valid(Hash) and Invalid(Hash) -->
This procedure takes in input the round $R$, the iteration $I$, and the candidate block $\mathsf{B}^c$ (as returned by [*ProposalStep*][props]) and outputs the Validation result $`(v^\mathsf{V}, \mathsf{SV}_{v^\mathsf{V}})`$, where $v^\mathsf{V}$ is $Valid$, $Invalid$, $NoCandidate$, or $NoQuorum$, and $\mathsf{SV}_{v^\mathsf{V}}$ is the aggregated vote of the quorum committee.

The procedure performs two tasks: 

1. If the node is part of the Validation committee $\mathcal{C}$, it verifies the candidate $\mathsf{B}^c$ and broadcasts a [`Validation`][vmsg] message with its vote: $Valid$ if $\mathsf{B}^c$ is a valid successor of the local $Tip$, $Invalid$ if it's not, and $NoCandidate$ if it's $NIL$ (no candidate has been received).
If $\mathsf{B}^c$'s parent is not $Tip$, it is discarded (it likely belongs to a fork)[^2].

2. It collects `Validation` messages from all committee members, and sets the result depending on the votes:
   - if $Valid$ votes reach $Supermajority$, the step outputs $Valid$;
   - if $Invalid$ votes reach $Majority$, the step outputs $Invalid$;
   - if $NoCandidate$ votes reach $Majority$, the step outputs $NoCandidate$;
   - if the timeout $\tau_{Val}$ expires, the step outputs $NoQuorum$.

Collected votes are aggregated in [`StepVotes`][sv] structures. In particular, for each vote $v$ ($Valid$ / $Invalid$ / $NoCandidate$ / $NoQuorum$), a $\mathsf{SV}_v=(\sigma_v,\boldsymbol{bs}_v)$ is used.

**Parameters**
- $R$: round number
- $I$: iteration number
- $\mathsf{B}^c$: candidate block

**Algorithm**
1. Extract committee $\mathcal{C}$ for the step ([*ExtractCommittee*][ec])
2. Start step timeout $\tau_{Val}$
3. If the node $\mathcal{N}$ is part of $\mathcal{C}$:
   1. If candidate $\mathsf{B}^c$ is empty
   2. or the previous block is not $Tip$:
      1. Set vote $v$ to $NoCandidate$
   3. Otherwise:
      1. Verify $\mathsf{B}^c$ against $Tip$
      2. If $\mathsf{B}^c$ is valid, set vote $v$ to $Valid$
      3. Otherwise, set $v$ to $Invalid$
   4. Set $Vote$ to $(v, \eta_{\mathsf{B}^c})$
   5. Create a $\mathsf{Validation}$ message $\mathsf{M}$ for vote $v$
   6. Broadcast $\mathsf{M}$

4. For each vote $v$ ($Valid$, $Invalid$, $NoCandidate$)
   1. Initialize $\mathsf{SV}_v$

5. While timeout $\tau_{Val}$ has not expired:
   1. If a $\mathsf{Validation}$ message $\mathsf{M^V}$ is received for round $R$ and iteration $I$:
      1. If $\mathsf{M^V}$'s signature is valid
      2. and $\mathsf{M^V}$'s signer is in the committee $\mathcal{C}$
      3. and $\mathsf{M^V}$'s vote is valid ($NoQuorum$, $Valid$, or $Invalid$)
         1. Propagate $\mathsf{M^V}$
         2. Collect $\mathsf{M^V}$'s vote $v^\mathsf{V}$ into the aggregated $\mathsf{SV}_{v^\mathsf{V}}$
         3. Set the target quorum $Q$ to $Supermajority$ if $v^\mathsf{V}$ is $Valid$, or to $Majority$ if $^V$ is $Invalid$ or $NoCandidate$
         4. If votes in $\mathsf{SV}_{v^\mathsf{V}}$ reach $Q$
            1. Store elapsed time
            2. Output $(v^\mathsf{V}, \mathsf{SV}_{v^\mathsf{V}})$

 6. If timeout $\tau_{Val}$ expired:
    1. Increase timeout
    2. Output $(NoQuorum, NIL)$

**Procedure**

$ValidationStep( R, I, \mathsf{B}^c ) :$
1. $\mathcal{C}=$ [*ExtractCommittee*][ec]$(R,I, ValStep)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } (pk_\mathcal{N} \in \mathcal{C}):$
   1. $\texttt{if } (\mathsf{B}^c = NIL)$
   2. $\texttt{or } (\mathsf{B}^c.PreviousBlock \ne Tip.Hash):$
      1. $v = NoCandidate$
   3. $\texttt{else}:$
      1. $isValid$ = [*VerifyBlockHeader*][vbh]$(Tip,\mathsf{B}^c)$
      2. $\texttt{if } (isValid = true) : v = Valid$
      3. $\texttt{else}: v = Invalid$
   4. $\texttt{set}: \mathsf{VI} = (v, \eta_{\mathsf{B}^c})$
   5. $`\mathsf{M} = `$ [*CMsg*][cmsg]$(\mathsf{Validation}, \mathsf{VI})$
      | Field           | Value                 | 
      |-----------------|-----------------------|
      | $PrevHash$      | $\eta_{Tip}$          |
      | $Round$         | $R$                   |
      | $Iteration$     | $I$                   |
      | $Vote$          | $v$                   |
      | $CandidateHash$ | $\eta_{\mathsf{B}^c}$ |
      | $Signer$        | $pk_\mathcal{N}$      |
      | $Signature$     | $\sigma_\mathsf{M}$   |

   6. [*Broadcast*][mx]$(\mathsf{M})$

4. $\texttt{set}:$
   - $\texttt{for } v \texttt{ in } [Valid, Invalid, NoCandidate]:$
     - $\mathsf{SV}_v = (\sigma_v, \boldsymbol{bs}_v)$

5. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Val}) \texttt{ and } (I \lt EmergencyMode):$
   1. $\texttt{if } (\mathsf{M^V} =$ [*Receive*][mx]$(\mathsf{Validation},R,I) \ne NIL):$
      - $\texttt{set}:$
        - $\mathsf{CI}, \mathsf{VI}, \mathsf{SI} \leftarrow \mathsf{M^V}$
        - $`\eta_{\mathsf{B}^p}, \_, \_, \leftarrow \mathsf{CI}`$
        - $v^\mathsf{V}, \eta_{\mathsf{B}^c} \leftarrow \mathsf{VI}$
        - $pk_\mathsf{M^V}, \sigma_\mathsf{M^V} \leftarrow \mathsf{SI}$

      1. $\texttt{if } (pk_{\mathsf{M^V}} \in \mathcal{C})$
      2. $\texttt{and }($[*VerifyMessage*][sigs]$(\mathsf{M^V}) = true)$
      3. $\texttt{and }(v^\mathsf{V} \in \{NoCandidate,Valid,Invalid\}):$
         1. [*Propagate*][mx]$(\mathsf{M^V})$
         2. $`\mathsf{SV}_{v^\mathsf{V}} =`$ [*AggregateVote*][av]$`( \mathsf{SV}_{v^\mathsf{V}}, \mathcal{C}, \sigma_\mathsf{M^V}, pk_\mathsf{M^V} )`$
         3. $Q =$ [*GetQuorum*][gq]$(v^\mathsf{V})$
         4. $\texttt{if }($[*countSetBits*][csb]$(\boldsymbol{bs}_{v^\mathsf{V}}) \ge Q):$
            1. [*StoreElapsedTime*][set]$(Validation, \tau_{Now}-\tau_{Start})$
            2. $\texttt{output } (v^\mathsf{V}, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^\mathsf{V}})$

 6. $\texttt{if } (\tau_{Now} \gt \tau_{Start}+\tau_{Val}):$
    1. [*IncreaseTimeout*][it]$(Validation)$
    2. $\texttt{output } (NoQuorum, NIL, NIL)$


<!----------------------- FOOTNOTES ----------------------->
[^1]: remember that the quorum is calculated over the weight of the voters, not their number.
[^2]: we do not consider a candidate block with the wrong parent as $Invalid$ because we do not want to punish the generator for being on a fork.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md -->
[vals]: #validation-step


<!-- Protocol -->
[env]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#environment
[set]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#storeelapsedtime
[it]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#increasetimeout
[sai]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#saiteration

[prop]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md
[props]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md#proposalstep

[rat]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md

[ds]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md
[dsp]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md#deterministic-sortition-ds


<!-- Basics -->
[vbh]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#verifyblockheader
[rf]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#rolling-finality

[p]:     https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#provisioners-and-stakes
[pen]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#penalties

[vc]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#voting-committees
[ec]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#ExtractCommittee
[gq]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetQuorum
[gsn]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetStepNum
[sc]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#subcommittees
[csb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#countsetbits
[sv]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#stepvotes
[av]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#aggregatevote


<!-- Messages -->
[sigs]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#signatures
[mx]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#procedures-1
[vmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#validation
[cmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#cmsg

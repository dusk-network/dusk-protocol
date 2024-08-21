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
This procedure takes in input the round $R$, the iteration $I$, and the candidate block $\mathsf{B}^c$ (as returned by [*ProposalStep*][props]) and outputs the Validation result $`(\mathsf{V}, \mathsf{SV_V})`$, where $\mathsf{V}$ is $Valid$, $Invalid$, $NoCandidate$, or $NoQuorum$, and $\mathsf{SV_V}$ is the aggregated signature of the quorum committee.

The procedure performs two tasks: 

1. If the node is part of the Validation committee $\mathcal{C}$, it verifies the candidate $\mathsf{B}^c$ and broadcasts a [`Validation`][vmsg] message with its vote: $Valid(\eta_{\mathsf{B}^c})$ if $\mathsf{B}^c$ is a valid successor of the local $Tip$, $Invalid((\eta_{\mathsf{B}^c}))$ if it's not, and $NoCandidate$ if no candidate has been received.
If $\mathsf{B}^c$'s parent is not $Tip$, it is discarded (it likely belongs to a fork)[^2].

1. It collects `Validation` messages from all committee members, and sets the result depending on the votes:
   - if $Valid$ votes reach $Supermajority$, the step outputs $Valid$;
   - if $Invalid$ votes reach $Majority$, the step outputs $Invalid$;
   - if $NoCandidate$ votes reach $Majority$, the step outputs $NoCandidate$;
   - if the timeout $\tau_{Val}$ expires, the step outputs $NoQuorum$.

Collected vote signatures are aggregated in [`StepVotes`][sv] structures. In particular, for each vote $\mathsf{V}$ ($Valid$ / $Invalid$ / $NoCandidate$ / $NoQuorum$), a $\mathsf{SV_V}=(\sigma_\mathsf{V},\boldsymbol{bs}_\mathsf{V})$ is used.

In the collection of votes, the $StepVoters$ set variable is used to track provisioners that voted for this step. Conflicting votes are discarded and can be liable to [slashing][pen].

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
      1. Set vote $\mathsf{V}$ to $NoCandidate$
   3. Otherwise:
      1. Verify $\mathsf{B}^c$ against $Tip$
      2. If $\mathsf{B}^c$ is valid, set vote $\mathsf{V}$ to $Valid(\eta_{\mathsf{B}^c})$
      3. Otherwise, set $\mathsf{V}$ to $Invalid(\eta_{\mathsf{B}^c})$
   4. Create a $\mathsf{Validation}$ message $\mathsf{M}$ for vote $\mathsf{V}$
   5. Broadcast $\mathsf{M}$

4. While timeout $\tau_{Val}$ has not expired:
   1. If a $\mathsf{Validation}$ message $\mathsf{M}$ is received for round $R$ and iteration $I$:
      1. If $\mathsf{M}$'s signature is valid
      2. and $\mathsf{M}$'s signer is in the committee $\mathcal{C}$
      3. and $\mathsf{M}$'s vote is valid ($NoQuorum$, $Valid$, or $Invalid$)
      4. and no other vote has been received for $\mathsf{M}$'s signer
         1. Propagate $\mathsf{M}$
         2. Collect $\mathsf{M}$'s vote $\mathsf{V}$ into the aggregated $\mathsf{SV_V}$
         3. Add $\mathsf{M}$'s signer to the $StepVoters$ set
         4. Set the target quorum $Q$ to $Supermajority$ if $\mathsf{V}$ is $Valid$, or to $Majority$ if $V$ is $Invalid$ or $NoCandidate$
         5. If votes in $\mathsf{SV_V}$ reach $Q$
            1. Store elapsed time
            2. Output $(\mathsf{V}, \mathsf{SV_V})$

 5. If timeout $\tau_{Val}$ expired:
    1. Increase timeout
    2. Output $(NoQuorum, NIL)$

**Procedure**

$ValidationStep( R, I, \mathsf{B}^c ) :$
1. $\mathcal{C}=$ [*ExtractCommittee*][ec]$(R,I, ValStep)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } (pk_\mathcal{N} \in \mathcal{C}):$
   1. $\texttt{if } (\mathsf{B}^c = NIL)$
   2. $\texttt{or } (\mathsf{B}^c.PreviousBlock \ne Tip.Hash):$
      1. $\mathsf{V} = NoCandidate$
   3. $\texttt{else}:$
      1. $isValid$ = [*VerifyBlockHeader*][vbh]$(Tip,\mathsf{B}^c)$
      2. $\texttt{if } (isValid = true) : \mathsf{V} = Valid(\eta_{\mathsf{B}^c})$
      3. $\texttt{else}: \mathsf{V} = Invalid(\eta_{\mathsf{B}^c})$
   4. $`\mathsf{M} = `$ [*CMsg*][cmsg]$(\mathsf{Validation}, \mathsf{V})$
      | Field           | Value                                 |
      |-----------------|---------------------------------------|
      | $ConsensusInfo$ | $(\eta_{Tip}, R, I)$                  |
      | $Vote$          | $\mathsf{V}$                          |
      | $SignInfo$      | $(pk_\mathcal{N}, \sigma_\mathsf{M})$ |

   5. [*Broadcast*][broad]$(\mathsf{M})$

4. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Val}) \texttt{ and } (I \lt EmergencyMode):$
   - $StepVoters = \emptyset$
   1. $\texttt{if } (\mathsf{M} =$ [*Receive*][recv]$(\mathsf{Validation},R,I) \ne NIL):$
      - $\texttt{set}:$
        - $\mathsf{CI}, \mathsf{V}, \mathsf{SI} \leftarrow \mathsf{M}$
        - $`\eta_{\mathsf{B}^p}, \_, \_, \leftarrow \mathsf{CI}`$
        - $pk_\mathsf{M}, \sigma_\mathsf{M} \leftarrow \mathsf{SI}$

      1. $\texttt{if } (pk_\mathsf{M} \in \mathcal{C})$
      2. $\texttt{and }($[*VerifyMessage*][sigs]$(\mathsf{M}) = true)$
      3. $\texttt{and }(\mathsf{V} \in \{NoCandidate,Valid,Invalid\})$
      4. $\texttt{and }(pk_\mathsf{M} \notin StepVoters):$
         1. [*Propagate*][propm]$(\mathsf{M})$
         2. $\mathsf{SV_V} =$ [*AggregateVote*][av]$`( \mathsf{SV_V}, \mathcal{C}, \sigma_\mathsf{M}, pk_\mathsf{M} )`$
         3. $StepVoters = StepVoters \cup pk_\mathsf{M}$
         4. $Q =$ [*GetQuorum*][gq]$(\mathsf{V})$
         5. $\texttt{if }($[*countSetBits*][csb]$(\boldsymbol{bs}_{\mathsf{V}}) \ge Q):$
            1. [*StoreElapsedTime*][set]$(Validation, \tau_{Now}-\tau_{Start})$
            2. $\texttt{output } (\mathsf{V}, \mathsf{SV_V})$

 5. $\texttt{if } (\tau_{Now} \gt \tau_{Start}+\tau_{Val}):$
    1. [*IncreaseTimeout*][it]$(Validation)$
    2. $\texttt{output } (NoQuorum, NIL)$


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
[vmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#validation
[cmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#cmsg
[broad]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#broadcast
[recv]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#receive
[propm]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#propagate

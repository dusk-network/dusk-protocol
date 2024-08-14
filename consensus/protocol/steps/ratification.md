# Ratification
*Ratification* is the third step in an [*SA iteration*][sai]. In this step, the result of the [Validation][val] step is agreed upon by another committee of randomly chosen provisioners.

Members of the extracted committee cast a vote with the output of the [Validation][val] step as resulting from the `Validation` messages received. At the same time, all provisioners, including committee members, collect Ratification votes from the network until a target quorum is reached or the step timeout expires.

If a quorum is reached for any result, a [`Quorum`][qmsg] message is generated with the aggregated signatures of both Validation and Ratification steps.
Since the attestation proves a candidate reached a quorum, receiving this message is sufficient to accept the candidate into the local chain.

The main purpose of the Ratification step is to ensure provisioners are "aligned" with respect to the Validation result: if Validation result was $Valid$, it ensures a supermajority of provisioners saw such a result and hence accepted the block. Similarly, in case of $\text{non-}Valid$ result, it ensures a majority of provisioners will certify this iteration as failed, which, in turn, is used in determining if a winning candidate will be Attested or not (see [Rolling Finality][rf]).

**ToC**
  - [Overview](#overview)
    - [Procedures](#procedures)
      - [*RatificationStep*](#ratificationstep)


## Overview
In the Ratification step, each node first executes the [*Deterministic Sortition*][ds] algorithm to extract the [Voting Committee][vc] for the step.

If the node is part of the committee, it casts a vote with the winning [Validation][val] vote ($Valid$, $Invalid$, $NoCandidate$, $NoQuorum$). 
The vote is broadcast using a [`Ratification`][rmsg] message, which includes the `StepVotes` of the aggregated Validation votes.

Then, all nodes, including the committee members, collect votes from the network until a *supermajority*  ($\frac{2}{3}$ of the committee credits) of $Valid$ votes is reached, a *majority* ($\frac{1}{2}{+}1$) of $\text{non-}Valid$ votes is reached, or the step timeout expires. 
Ratification votes (except for $NoQuorum$) are only considered valid if accompanied by a [`StepVotes`][sv] with a quorum of matching Validation votes.

Note that, since Ratification votes require a quorum of Validation votes (except for $NoQuorum$) to be valid, only one vote between $Valid$, $Invalid$, and $NoCandidate$ can be received, plus $NoQuorum$. In other words, $Valid$, $Invalid$, and $NoCandidate$ are mutually exclusive in Ratification. In fact, it's impossible to receive a valid $NoCandidate$ vote (which requires a majority) and $Valid$ vote in the same Ratification step, unless a provisioner casts a double vote[^1].

If any quorum is reached, the step outputs the winning vote ($Valid$, $Invalid$, $NoCandidate$). Otherwise, if the step timeout expires, the step outputs $NoQuorum$.
In all cases, except $NoQuorum$, the output of the step includes a [`StepVotes`][sv] structure with the aggregated votes that determined the result.

The output, together with the Validation output, will be used to determine the outcome of the iteration.


### Procedures

#### *RatificationStep*
This procedure takes in input the round $R$, the iteration $I$, and the Validation result $\mathsf{SR}^V$ (as returned from [*ValidationStep*][vals]) and outputs the Ratification result $`\mathsf{SR}^R=(\mathsf{V}^R, \mathsf{SV}_{\mathsf{V}^R})`$, where $\mathsf{V^R}$ is $Valid$, $Invalid$, $NoCandidate$, or $NoQuorum$, and $\mathsf{SV}_{\mathsf{V}^R}$ is the aggregated signature of the quorum committee.

The procedure performs two tasks: 

1. If the node is part of the Ratification committee $\mathcal{C}$, it broadcasts a [`Ratification`][rmsg] message with the Validation result $(\mathsf{V}^V, \mathsf{SV}_{\mathsf{V}^V})$.

2. It collects `Ratification` messages from all committee members, and sets the result depending on the votes:
   - if $Valid$ votes reach $Supermajority$, the step outputs $Valid$;
   - if $Invalid$ votes reach $Majority$, the step outputs $Invalid$;
   - if $NoCandidate$ votes reach $Majority$, the step outputs $NoCandidate$;
   - if the timeout $\tau_{Rat}$ expires, the step outputs $NoQuorum$.

Collected vote signatures are aggregated in [`StepVotes`][sv] structures. In particular, for each vote $\mathsf{V}$ ($Valid$ / $Invalid$ / $NoCandidate$ / $NoQuorum$), a $\mathsf{SV_V}=(\sigma_\mathsf{V},\boldsymbol{bs}_\mathsf{V})$ is used.

**Parameters**
- $R$: round number
- $I$: iteration number
- $\mathsf{SR}^V$ : [`StepResult`][sr] of the previous Validation step

**Algorithm**
1. Extract committee $\mathcal{C}$ for the step ([*ExtractCommittee*][ec])
2. Start step timeout $\tau_{Rat}$
3. If the node $\mathcal{N}$ is part of $\mathcal{C}$:
   1. Create a $\mathsf{Ratification}$ message $\mathsf{M}$ for vote $\mathsf{V}$
   2. Broadcast $\mathsf{M}$

4. While timeout $\tau_{Rat}$ has not expired:
   1. If a $\mathsf{Ratification}$ message $\mathsf{M}$ is received for round $R$ and iteration $I$:
      1. If $\mathsf{M}$'s signature is valid
      2. and $\mathsf{M}$'s signer is in the committee $\mathcal{C}$
      3. and $\mathsf{M}$'s vote is valid ($NoQuorum$, $Valid$, $Invalid$, or $NoQuorum$)
      4. and $ValidationVotes$ is a valid quorum of votes for the previous Validation step :
         1. Propagate $\mathsf{M}$
         2. Collect $\mathsf{M}$'s vote $\mathsf{V}$ into the aggregated $\mathsf{SV_V}$
         3. Set the target quorum $Q$ to $Supermajority$ if $\mathsf{V}$ is $Valid$, or to $Majority$ otherwise
         4. If votes in $\mathsf{SV_V}$ reach $Q$
            1. Store elapsed time
            2. Output $(\mathsf{V}, \mathsf{SV_V})$

 5. If timeout $\tau_{Rat}$ expired:
    1. Increase timeout
    2. Output $(NoQuorum, NIL)$

**Procedure**

$RatificationStep( R, I, \mathsf{SR}^V ) :$
- $\texttt{set}:$ 
  - $\mathsf{V}, \mathsf{SV_V} \leftarrow \mathsf{SR}^V$
1. $\mathcal{C}=$ [*ExtractCommittee*][ec]$(R,I, RatStep)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } (pk_\mathcal{N} \in \mathcal{C}):$
   1. $`\mathsf{M} = `$ [*CMsg*][cmsg]$`(\mathsf{Ratification}, \mathsf{V}, \mathsf{SV_V}, \tau_{Now})`$
      | Field             | Value                                 |
      |-------------------|---------------------------------------|
      | $ConsensusInfo$   | $(\eta_{Tip}, R, I)$                  |
      | $Vote$            | $\mathsf{V}$                          |
      | $ValidationVotes$ | $\mathsf{SV_V}$                       |
      | $Timestamp$       | $\tau_{Now}$                          |
      | $SignInfo$        | $(pk_\mathcal{N}, \sigma_\mathsf{M})$ |

   2. [*Broadcast*][mx]$(\mathsf{M})$

4. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Rat}) \texttt{ and } (I \lt EmergencyMode):$
   1. $\texttt{if } (\mathsf{M} =$ [*Receive*][mx]$(\mathsf{Ratification},R,I) \ne NIL):$
      - $\texttt{set}:$
        - $`\mathsf{CI}, \mathsf{V}, \mathsf{SV}^V, \_, \mathsf{SI} \leftarrow \mathsf{M}`$
        - $`\eta_{\mathsf{B}^p}, \_, \_, \leftarrow \mathsf{CI}`$
        - $`pk_\mathsf{M}, \sigma_\mathsf{M} \leftarrow \mathsf{SI}`$
        - $\upsilon^V = (\mathsf{CI}||\mathsf{V}||ValStep)$
        - $Q^V =$ [*GetQuorum*][gq]$(\mathsf{V})$

      1. $\texttt{if } (pk_\mathsf{M} \in \mathcal{C})$
      2. $\texttt{and }($[*VerifyMessage*][sigs]$(\mathsf{M}) = true)$
      3. $\texttt{and }(\mathsf{V} \in \{NoCandidate,Valid,Invalid,NoQuorum\})$
      4. $\texttt{and }($[*VerifyVotes*][vv]$(\mathsf{SV}^V, \upsilon^V, Q^V, \mathcal{C}^V) = true):$
         1. [*Propagate*][mx]$(\mathsf{M})$
         2. $`\mathsf{SV_V} =`$ [*AggregateVote*][av]$`( \mathsf{SV_V}, \mathcal{C}, \sigma_\mathsf{M}, pk_{\mathsf{M}} )`$
         3. $Q =$ [*GetQuorum*][gq]$(\mathsf{V})$
         4. $\texttt{if }($[*countSetBits*][csb]$(\boldsymbol{bs}_{\mathsf{V}}) \ge Q):$
            1. [*StoreElapsedTime*][set]$(Ratification, \tau_{Now}-\tau_{Start})$
            2. $\texttt{output } (\mathsf{V}, \mathsf{SV_V})$

 5. $\texttt{if } (\tau_{Now} \gt \tau_{Start}+\tau_{Rat}):$
    1. [*IncreaseTimeout*][it]$(Ratification)$
    2. $\texttt{output } (NoQuorum, NIL)$


<!----------------------- FOOTNOTES ----------------------->
[^1]: We here implicitly assume no double votes are cast. However, such a behavior is subject to [slashing][pen].

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md -->
[rs]: #ratificationstep

<!-- Basics -->
[pen]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#penalties
[p]:     https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#provisioners-and-stakes

[vc]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#voting-committees
[ec]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#ExtractCommittee
[gq]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetQuorum
[gsn]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetStepNum
[sc]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#subcommittees
[csb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#countsetbits
[sr]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#stepresult
[sv]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#stepvotes
[av]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#aggregatevote
[vv]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#verifyvotes

[vbh]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#verifyblockheader
[rf]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#rolling-finality

<!-- Protocol -->
[env]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#environment
[set]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#storeelapsedtime
[it]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#increasetimeout
[sai]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#saiteration

[prop]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md
[props]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md#proposalstep

[val]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md
[vals]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md#validation-step

[ds]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md
[dsp]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md#deterministic-sortition-ds

<!-- Messages -->
[sigs]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#signatures
[rmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#ratification
[qmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#quorum
[cmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#cmsg
[mx]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#procedures-1
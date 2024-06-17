# Ratification
*Ratification* is the third step in an [*SA iteration*][sai]. In this step, the result of the [Validation][val] step is agreed upon by another committee of randomly chosen provisioners.

Members of the extracted committee cast a vote with the output of the [Validation][val] step as resulting from the `Validation` messages received. At the same time, all provisioners, including committee members, collect Ratification votes from the network until a target quorum is reached or the step timeout expires.

If a quorum is reached for any result, a [`Quorum`][qmsg] message is generated with the aggregated signatures of both Validation and Ratification steps.
Since the attestation proves a candidate reached a quorum, receiving this message is sufficient to accept the candidate into the local chain.

The main purpose of the Ratification step is to ensure provisioners are "aligned" with respect to the Validation result: if Validation result was $Valid$, it ensures a supermajority of provisioners saw such a result and hence accepted the block. Similarly, in case of $\text{non-}Valid$ result, it ensures a majority of provisioners will certify this iteration as failed, which, in turn, is used in determining if a winning candidate will be Attested or not (see [Finality][fin]).

### ToC
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
This procedure takes in input the round $R$, the iteration $I$, and the Validation result $\mathsf{SR}^V$ (as returned from [*ValidationStep*][vals]) and outputs the Ratification result $`\mathsf{SR}^R=(v^\mathsf{R}, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^\mathsf{R}})`$, where $v^\mathsf{R}$ is $Valid$, $Invalid$, $NoCandidate$, or $NoQuorum$, $\eta_{\mathsf{B}^c}$ is the candidate hash, and $\mathsf{SV}_{v^\mathsf{R}}$ is the aggregated vote of the quorum committee.

The procedure performs two tasks: 

1. If the node is part of the Ratification committee $\mathcal{C}$, it broadcasts a `Ratification` message with the Validation result $(v^\mathsf{V}, \mathsf{SV}_{v^\mathsf{V}})$.

2. It collects `Ratification` messages from all committee members, and sets the result depending on the votes:
   - if $Valid$ votes reach $Supermajority$, the step outputs $Valid$;
   - if $Invalid$ votes reach $Majority$, the step outputs $Invalid$;
   - if $NoCandidate$ votes reach $Majority$, the step outputs $NoCandidate$;
   - if the timeout $\tau_{Rat}$ expires, the step outputs $NoQuorum$.

Collected votes are aggregated in [`StepVotes`][sv] structures. In particular, for each vote $v$ ($Valid$ / $Invalid$ / $NoCandidate$ / $NoQuorum$), a $\mathsf{SV}_v=(\sigma_v,\boldsymbol{bs}_v)$ is used.

**Parameters**
- $R$: round number
- $I$: iteration number
- $\mathsf{SR}^V = (v^\mathsf{V}, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^\mathsf{V}})$ Validation result ([`StepResult`][sr])

**Algorithm**
1. Extract committee $\mathcal{C}$ for the step ([*ExtractCommittee*][ec])
2. Start step timeout $\tau_{Rat}$
3. If the node $\mathcal{N}$ is part of $\mathcal{C}$:
   1. Create a $\mathsf{Ratification}$ message $\mathsf{M}$ for vote $v^\mathsf{V}$
   2. Broadcast $\mathsf{M}$

4. For each vote $v$ ($Valid$, $Invalid$, $NoCandidate$, $NoQuorum$)
   1. Initialize $\mathsf{SV}_v$

5. While timeout $\tau_{Rat}$ has not expired:
   1. If a $\mathsf{Ratification}$ message $\mathsf{M^R}$ is received for round $R$ and iteration $I$:
      1. If $\mathsf{M^R}$'s signature is valid
      2. and $\mathsf{M^R}$'s signer is in the committee $\mathcal{C}$
      3. and $\mathsf{M^R}$'s vote is valid ($NoQuorum$, $Valid$, $Invalid$, or $NoQuorum$)
      4. and $ValidationVotes$ is a valid quorum of votes for the previous Validation step :
         1. Propagate $\mathsf{M^R}$
         2. Collect $\mathsf{M^R}$'s vote $v^\mathsf{R}$ into the aggregated $\mathsf{SV}_{v^\mathsf{R}}$
         3. Set the target quorum $Q$ to $Supermajority$ if $v^\mathsf{R}$ is $Valid$, or to $Majority$ otherwise
         4. If votes in $\mathsf{SV}_{v^\mathsf{R}}$ reach $Q$
            1. Store elapsed time
            2. Output $(v^\mathsf{R}, \mathsf{SV}_{v^\mathsf{R}})$

 6. If timeout $\tau_{Rat}$ expired:
    1. Increase timeout
    2. Output $(NoQuorum, NIL)$

**Procedure**

$RatificationStep( R, I, \mathsf{SR}^V ) :$
- $\texttt{set}:$ 
  - $v^\mathsf{V}, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^\mathsf{V}} \leftarrow \mathsf{SR}^V$
1. $\mathcal{C}=$ [*ExtractCommittee*][ec]$(R,I, RatStep)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } (pk_\mathcal{N} \in \mathcal{C}):$
   1. $`\mathsf{M} = `$ [*Msg*][msg]$`(\mathsf{Ratification}, v^\mathsf{V}, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^\mathsf{V}}, \tau_{Now})`$
      | Field             | Value                        | 
      |-------------------|------------------------------|
      | $PrevHash$        | $\eta_{Tip}$                 |
      | $Round$           | $R$                          |
      | $Iteration$       | $I$                          |
      | $Vote$            | $v^\mathsf{V}$               |
      | $CandidateHash$   | $\eta_{\mathsf{B}^c}$        |
      | $ValidationVotes$ | $\mathsf{SV}_{v^\mathsf{V}}$ |
      | $Timestamp$       | $\tau_{Now}$                 |
      | $Signer$          | $pk_\mathcal{N}$             |
      | $Signature$       | $\sigma_\mathsf{M}$          |

   2. [*Broadcast*][mx]$(\mathsf{M})$

4. $\texttt{set}:$
   - $\texttt{for } v \texttt{ in } [Valid, Invalid, NoCandidate, NoQuorum]:$
     - $\mathsf{SV}_v = (\sigma_v, \boldsymbol{bs}_v)$

5. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Rat}) \texttt{ and } (I \lt EmergencyMode):$
   1. $\texttt{if } (\mathsf{M^R} =$ [*Receive*][mx]$(\mathsf{Ratification},R,I) \ne NIL):$
      - $\texttt{set}:$
        - $`\mathsf{CI}, \mathsf{VI}, \mathsf{SV}^V, \_, \mathsf{SI} \leftarrow \mathsf{M^R}`$
        - $`\eta_{\mathsf{B}^p}, \_, \_, \leftarrow \mathsf{CI}`$
        - $`v^\mathsf{R}, \eta_{\mathsf{B}^c} \leftarrow \mathsf{VI}`$
        - $`pk_\mathsf{M^R}, \sigma_\mathsf{M^R} \leftarrow \mathsf{SI}`$
        - $\upsilon^V = (\mathsf{CI}||\mathsf{VI}||ValStep)$
        - $Q^V =$ [*GetQuorum*][gq]$(v^\mathsf{R})$

      1. $\texttt{if } (pk_\mathsf{M^R} \in \mathcal{C})$
      2. $\texttt{and }($[*VerifyMessage*][ms]$(\mathsf{M^R}) = true)$
      3. $\texttt{and }(v^\mathsf{R} \in \{NoCandidate,Valid,Invalid,NoQuorum\})$
      4. $\texttt{and }($[*VerifyVotes*][vv]$(\mathsf{SV}^V, \upsilon^V, Q^V, \mathcal{C}^V) = true):$
         1. [*Propagate*][mx]$(\mathsf{M^R})$
         2. $`\mathsf{SV}_{v^\mathsf{R}} =`$ [*AggregateVote*][av]$`( \mathsf{SV}_{v^\mathsf{R}}, \mathcal{C}, \sigma_\mathsf{M^R}, pk_{\mathsf{M^R}} )`$
         3. $Q =$ [*GetQuorum*][gq]$(v^\mathsf{R})$
         4. $\texttt{if }($[*countSetBits*][cb]$(\boldsymbol{bs}_{v^\mathsf{R}}) \ge Q):$
            1. [*StoreElapsedTime*][set]$(Ratification, \tau_{Now}-\tau_{Start})$
            2. $\texttt{output } (v^\mathsf{R}, \eta_{\mathsf{B}^c}, \mathsf{SV}_{v^\mathsf{R}})$

 6. $\texttt{if } (\tau_{Now} \gt \tau_{Start}+\tau_{Rat}):$
    1. [*IncreaseTimeout*][it]$(Ratification)$
    2. $\texttt{output } (NoQuorum, NIL, NIL)$


<!----------------------- FOOTNOTES ----------------------->
[^1]: We currently assume no double votes are cast. In a future version of the protocol, double votes will be punished with [slashing][sla].

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md -->
[rs]: #ratificationstep

<!-- Basics -->
[sla]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#slashing
[vc]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#voting-committees
[ec]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#ExtractCommittee
[sc]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#subcommittees
[cb]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#countsetbits
[sr]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#stepresult
[sv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#stepvotes

<!-- Consensus -->
[env]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#environment
[p]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#provisioners-and-stakes
[av]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#aggregatevote
[set]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#storeelapsedtime
[it]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#increasetimeout
[sai]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#saiteration
[gq]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#GetQuorum
[gsn]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#GetStepNum

<!-- Proposal -->
[prop]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal/README.md
[props]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal/README.md#proposalstep

<!-- Validation -->
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md
[vals]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md#validation-step

<!-- Sortition -->
[ds]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md
[dsp]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#deterministic-sortition-ds

<!-- Chain Management -->
[vbh]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#verifyblockheader
[vv]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#verifyvotes
[fin]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#finality
[rf]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#rolling-finality

<!-- Messages -->
[ms]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#signatures
[vmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#validation
[rmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#ratification
[qmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#quorum
[msg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#msg
[mx]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#procedures
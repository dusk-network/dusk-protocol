# Ratification
When the two [Reduction][red] votes reach a quorum, an [Agreement][amsg] message is produced, containing a [Certificate][cert] with the aggregated votes of the two quorum committees.
Since the certificate proves a candidate reached a quorum, this message is sufficient to accept the voted candidate into the local chain.

The *Ratification* task manages [Agreement][amsg] messages produced by the node or received from the network. In particular, when such a message is received (or produced), the corresponding candidate block is accepted ([*MakeWinning*](#makewinning)).

### ToC
- [Overview](#overview)
- [`Certificate`](#certificate)
- [Ratification Task](#ratification-task)
  - [*VerifyAgreement*](#verifyagreement)
  - [*VerifyAggregated*](#verifyaggregated)
  - [*CreateAggrAgreement*](#createaggragreement)
  - [*GetSignatures*](#getsignatures)
  - [*GetSigners*](#getsigners)
  - [*MakeWinning*](#makewinning)

## Overview
When a node generates or receives an $\mathsf{Agreement}$ message, it adds the included $\mathsf{Certificate}$ to the candidate block and make it a *winning block* for the corresponding round and iteration. 

If the $\mathsf{Agreement}$ message was received from the network, the aggregated votes ([StepVotes][sv]) are verified against the generator and committee members of the corresponding round and iteration. Both the validity of the votes and the quorum quota is verified.

The winning block will then be accepted as the new tip of the local chain.

## Certificate
The $\mathsf{Certificate}$ structure contains the [Reduction][rmsg] votes of the [quorum committee][sc] that reached consensus on a candidate block.
It is composed of two $\mathsf{StepVotes}$ structures, one for each [Reduction][red] step.


| Field             | Type            | Size     | Description                          |
|-------------------|-----------------|----------|--------------------------------------|
| $FirstReduction$  | [StepVotes][sv] | 56 bytes | Aggregated votes of first reduction  |
| $SecondReduction$ | [StepVotes][sv] | 56 bytes | Aggregated votes of second reduction |

The $\mathsf{Certificate}$ structure has a total size of 112 bytes.

## Ratification Procedure
The Ratification procedure receives and manages $\mathsf{Agreement}$ messages for a specific round and iteration. If the message was received from the network, it is first verified ([VerifyAgreement][va]) and then propagated.
The corresponding candidate block is then marked as the winning block of the round. 

***Parameters***
- [Consensus Parameters][cparams]

***Algorithm***
1. Loop:
   1. If an $\mathsf{Agreement}$ message $\mathsf{M}^A$ is received for the current round and iteration:
      1. Verify $\mathsf{M}^A$ is valid
      2. If valid:
         1. Propagate $\mathsf{M}^A$
         2. Create winning block $\mathsf{B}^w$ with $\mathsf{M}^A.Certificate$

***Procedure***

$\boldsymbol{\textit{Ratification}}( )$:
1. $\texttt{loop}$:   
   1.  $\texttt{if } (\mathsf{M}^A =$ [*Receive*][mx]$(\mathsf{Agreement}, r)):$
       - $`(\mathsf{H}_{\mathsf{M}^A}, \_ , \boldsymbol{bs}, \sigma_A) \leftarrow \mathsf{M}^A`$
       - $`\_, \mathsf{B}^c, r_{\mathsf{M}^A}, s_{\mathsf{M}^A} \leftarrow \mathsf{H}_{\mathsf{M}^A}`$
       1. [VerifyAggregated][vaa]$(\eta_{\mathsf{B}^c}, r_{\mathsf{M}^A}, s_{\mathsf{M}^{Ag}}, \boldsymbol{bs}, \sigma_{\boldsymbol{bs}})$
       2. [*Propagate*][mx]$(\mathsf{M}^A)$
       3. $\mathsf{B}^w =$ [*MakeWinning*][mw]$(\mathsf{M}^A.Agreement)$


### VerifyAgreement
*VerifyAgreement* verifies the validity of an $\mathsf{Agreement}$ message.

***Parameters***
- $\mathsf{M}$: $\mathsf{Agreement}$ message

***Algorithm***
1. Verify message signature
2. Check both Reduction votes are included
3. Check message iteration is lower than $MaxIterations$
4. Verify Reduction votes
5. If votes are valid, return true

***Procedure***
$VerifyAgreement(\mathsf{M})$
- $`\mathsf{H_M}, \sigma_{\mathsf{M}}, \boldsymbol{V} \leftarrow \mathsf{M}`$
- $`\_, r_{\mathsf{M}}, s_{\mathsf{M}}, \mathsf{B}^c \leftarrow \mathsf{H_M}`$
- $maxSteps = MaxIterations \times 3$
1. $Verify_{BLS}(\mathsf{H_M}, \sigma_{\mathsf{M}}, pk_{\mathsf{M}})$
2. $\texttt{if } (|\boldsymbol{V}| \ne 2): \texttt{output } false$
3. $\texttt{if } (s_{\mathsf{M}} \gt maxSteps): \texttt{output } false$
4. $`\mathsf{V}_1, \mathsf{V}_1 \leftarrow \boldsymbol{V}`$ \
  $r_1 =$ [*VerifyAggregated*][vaa]$`(\eta_{\mathsf{B}^c}, r_{\mathsf{M}}, s_{\mathsf{M}}-1, \boldsymbol{bs}_{\mathsf{V}_1}, \sigma_{\mathsf{V}_1})`$ \
  $r_2 =$ [*VerifyAggregated*][vaa]$`(\eta_{\mathsf{B}^c}, r_{\mathsf{M}}, s_{\mathsf{M}}, \boldsymbol{bs}_{\mathsf{V}_2}, \sigma_{\mathsf{V}_2})`$
1. $\texttt{if } (r_1{=}true) \texttt{ and } (r_2{=}true) : \texttt{return } true$

### VerifyAggregated
$VerifyAggregated$ checks the aggregated vote reaches the quorum, and the aggregated signature is valid for a specific block's hash, round, and step.

***Parameters***
- $hash$: the block's hash
- $round$: round
- $step$: step
- $\boldsymbol{bs}$: bitset of voters
- $\sigma_{\boldsymbol{bs}}$: aggregated signature

***Algorithm***
1. Compute subcommittee $C^{\boldsymbol{bs}}$ from bitset
2. If credits in subcommittee are less than $Quorum$
   1. Output false
3. Aggregate public keys of subcommittee member
4. Compute hash of candidate block, round, and step
5. Verify aggregated signature over hash

***Procedure***

$VerifyAggregated(hash, round, step, \boldsymbol{bs}, \sigma_{\boldsymbol{bs}})$:
1. $C^{\boldsymbol{bs}} = $[SubCommittee][sc]$(C_{round}^{step}, \boldsymbol{bs})$
2. $\texttt{if } ($[*CountCredits*][cc]$(C_{round}^{step}, C^{\boldsymbol{bs}}) \lt Quorum):$
   1. $\texttt{output } false$
3. $pk_{\boldsymbol{bs}} = AggregatePKs(C^{\boldsymbol{bs}})$
4. $\eta = Hash_{Blake2B}(round || step || hash)$
5. $\texttt{output } Verify_{BLS}(\eta, pk_{\boldsymbol{bs}}, \sigma_{\boldsymbol{bs}})$


### CreateAggrAgreement
*CreateAggrAgreement* aggregates all the signatures of a set $\mathsf{Agreement}$ messages, generates an $\mathsf{AggrAgreement}$ message, and broadcasts it.

***Parameters***
- $S = \{\mathsf{M}_1,..., \mathsf{M}_n\}$: vector of messages

***Algorithm***
1. Aggregate signatures
2. Create bitset
3. Create $\mathsf{AggrAgreement}$ message $\mathsf{M}$
4. Output message

***Procedure***

$CreateAggrAgreement(S)$:
 - $`\mathsf{H}_{\mathsf{M}_1}, \sigma_{\mathsf{M}_1}, \boldsymbol{V} \leftarrow \mathsf{M}_1`$
 - $`\_, r, s, \_ \leftarrow \mathsf{H}_{\mathsf{M}_1}`$
 1. $`\boldsymbol{sigs} = `$ [*GetSignatures*][gss]$(S)$ \
    $\sigma_{S} = Aggregate_{BLS}(\boldsymbol{sigs})$
 2. $`\boldsymbol{signers} = `$ [*GetSigners*][gs]$(S)$ \
    $`\boldsymbol{bs}_{S} = `$ [*BitSet*][bs]$(C_r^s, \boldsymbol{signers})$
 3. $`\mathsf{M}^{Ag} = (\mathsf{H}_{\mathsf{M}_1}, \sigma_{\mathsf{M}_1}, \boldsymbol{V}, \boldsymbol{bs}_{S}, \sigma_S)`$
 4. $\texttt{output }\mathsf{M}^{Ag}$

### GetSignatures
*GetSignatures* takes a list of messages and returns a vector of the message signatures.

***Parameters***

- $S = \{\mathsf{M}_1,..., \mathsf{M}_n\}$: storage of messages

***Procedure***

$GetSignatures(S):$
- $\boldsymbol{sigs} = []$
1. $\texttt{for } i = 0 \dots n :$
   1. $\boldsymbol{sigs}[i]=\mathsf{M}_i.Signature$
2. $\texttt{output } \boldsymbol{sigs}$


### GetSigners
*GetSigners* gets a storage of messages and returns a vector of the message signers.

***Parameters***

- $S = \{\mathsf{M}_1, \dots, \mathsf{M}_n\}$: storage of messages

***Procedure***

$GetSigners(S):$
- $\boldsymbol{signers} = []$
1. $\texttt{for } i = 0 \dots n :$
   1. $\boldsymbol{signers}[i]=\mathsf{M}^A_i.Signer$
2. $\texttt{output } \boldsymbol{signers}$


### MakeWinning
*MakeWinning* creates a certificate and adds it to the candidate block.

***Parameters***

- $\mathsf{B}^c$: candidate block to make winning
- $\mathsf{V}_1$: first-Reduction votes
- $\mathsf{V}_2$: second-Reduction votes

***Algorithm***

1. Create $Certificate$ $\Phi$ from first and second reduction votes
2. Add $\Phi$ to candidate block to create a winning block $\mathsf{B}^w$
3. Output $\mathsf{B}^w$

***Procedure***

$MakeWinning(\mathsf{B}^c, \mathsf{V}_1, \mathsf{V}_2):$
1. $\Phi = (\mathsf{V}_1, \mathsf{V}_2)$
2. $\mathsf{B}^c.Certificate = \Phi$
3. $\texttt{output } \mathsf{B}^c$

<!------------------------- LINKS ------------------------->

[cert]: #certificate-structure
[caa]:  #createaggragreement
[gs]:   #getsigners
[gss]:  #getsignatures
[mw]:   #makewinning
[va]:   #verifyagreement
[vaa]:  #verifyaggregated

<!-- Consensus -->
[cparams]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#parameters
<!-- Sortition -->
[bs]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#bitset
[cc]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#countcredits
[sc]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#subcommittee
<!-- Reduction -->
[red]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md 
[sv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md#stepvotes
<!-- Messages -->
[msg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-creation
[mx]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-exchange
[mh]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-header
[amsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#agreement-message
[aamsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#aggragreement-message
[rmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#reduction-message
[nbmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#newblock-message
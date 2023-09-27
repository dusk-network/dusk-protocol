# Ratification Phase
The *Ratification* phase certifies that a consensus has been reached on a candidate block and that a quorum of the last reduction committee is aware of that. 

All network nodes participate to this phase, collecting $\mathsf{Agreement}$ messages from the network, and aggregating them into an $\mathsf{AggrAgreement}$ message to be broadcast to other nodes as soon as enough of them have been received.

Each $\mathsf{Agreement}$ message ensures a provisioner of the committee received a quorum of votes for the candidate block and is then aware of the reached consensus.

### ToC
- [Overview](#overview)
  - [AggrAgreement Message](#aggragreement-message)
  - [Certificate](#certificate)
- [Ratification Algorithm](#ratification-algorithm)
  - [VerifyAgreement](#verifyagreement)
  - [VerifyAggregated](#verifyaggregated)
  - [CreateAggrAgreement](#createaggragreement)
  - [GetSignatures](#getsignatures)
  - [GetSigners](#getsigners)
  - [MakeWinning](#makewinning)

## Overview
During the Ratification phase, each node collects all $\mathsf{Agreement}$ messages coming from members of the second Reduction committee of the current round and iteration.

Each $\mathsf{Agreement}$ message is counted as many times as the influence (voting credits) of the member that created it.

When a node collects a quorum of $\mathsf{Agreement}$ messages, with respect to the committee of the second Reduction, it aggregates all the signatures of such messages and broadcast an $\mathsf{AggrAgreement}$ message containing one of the received $\mathsf{Agreement}$ messages and the aggregated signature over it.

When a node generates or receives an $\mathsf{AggrAgreement}$ message, it also produces a [Certificate][cert] for the candidate block, which contains the votes of the two Reduction committees (as per one $\mathsf{Agreement}$ message) and adds it to the block. 

When this occurs, the candidate block is tagged as *winning block* and accepted as the new chain tip, ending the current round.

### AggrAgreement Message

| Field           | Type                  | Size      | Description                                |
|-----------------|-----------------------|-----------|--------------------------------------------|
| $Header$        | [*MessageHeader*][mh] | 137 bytes | Agreement Message header                   |
| $Signature$     | BLS Signature         | 48 bytes  | Agreement Message signature                |
| $RVotes$        | [StepVotes][sv][ ]    | 112 bytes | First and second Reduction votes           |
| $Signers$       | BitSet                | 32 bits   | Bitset of aggregated signature             |
| $AggrSignature$ | BLS Signature         | 48 bytes  | Aggregated signature of Agreement's header |

The $\mathsf{AggrAgreement}$ message has a total size of 349 bytes

### Certificate

| Field             | Type            | Size     | Description                          |
|-------------------|-----------------|----------|--------------------------------------|
| $FirstReduction$  | [StepVotes][sv] | 56 bytes | Aggregated votes of first reduction  |
| $SecondReduction$ | [StepVotes][sv] | 56 bytes | Aggregated votes of second reduction |

The $Certificate$ structure has a total size of 112 bytes.

## Ratification Algorithm
The Ratification procedure collects $\mathsf{Agreement}$ messages for the current round until a specific candidate block reaches a quorum of such messages (with respect to the Second Reduction committee). When this occurs, an $\mathsf{AggrAgreement}$ message is broadcast, which contains the aggregated signatures from the collected $\mathsf{Agreement}$ messages, and the corresponding candidate block is marked as the winning block of the round. 
Receiving an $\mathsf{AggrAgreement}$ message from the network produces the same effect.

***Parameters***
- [Consensus Parameters][cparams]

**Environment**
- $S_i = \{\mathsf{M}^A_1,..., \mathsf{M}^A_n\}$ : Agreement message storage ($i$ is an iteration number, and $\mathsf{M}^A$ is an Agreement message for iteration $i$).

***Algorithm***
1. Infinite Loop:
   1. If an $\mathsf{Agreement}$ message $\mathsf{M}^A$ for round $r$ is received:
      1. If the signer is in the committee of the $\mathsf{M}^A$'s $Round$ and $Step$:
         1. Check $\mathsf{M}^A$'s validity
         2. Propagate $\mathsf{M}^A$
         3. If $\mathsf{M}^A$ is valid:
            1. Collect $\mathsf{M}^A$
            2. Count $\mathsf{Agreement}$ messages for $\mathsf{M}^A$'s round/iteration
            3. If a quorum has been reached
               1. Create $\mathsf{AggrAgreement}$ message $\mathsf{M}^{Ag}$
               2. Broadcast $\mathsf{M}^{Ag}$
               3. Create winning block $\mathsf{B}^w$
   2. If an $\mathsf{AggrAgreement}$ message $\mathsf{M}^{Ag}$ is received:
      1. Check if $\mathsf{M}^{Ag}$ is valid
      2. If valid:
         1. Propagate $\mathsf{M}^{Ag}$
         2. Create winning block $\mathsf{B}^w$

***Procedure***
$\boldsymbol{\textit{Ratification}}( )$:
1. $\texttt{loop}$:
   1. $\texttt{if } (\mathsf{M}^A =$ [*Receive*][mx]$(\mathsf{Agreement}, Round_{SA})):$
       - $s = Step_{\mathsf{M}^A}$
      1. $\texttt{if } (pk_{\mathsf{M}^A} \in C_r^s):$
          1. [*Propagate*][mx]$(\mathsf{M}^A)$
          2. $isValid$ = [*VerifyAgreement*][va]$(\mathsf{M}^A)$
          3. $\texttt{if } (isValid = true)$:
             1. $i = s \div 3$
             2. $S_i = S_i \cup \mathsf{M}^A$
             3. $\boldsymbol{signers}_i =$ [GetSigners][gs]$(S_i)$ \
                $c_i$ = [*CountCredits*][cc]$(C_r^s, \boldsymbol{signers}_i)$
             4. $\texttt{if } (c_{i} \ge Quorum):$
                1. $\mathsf{M}^{Ag}$ = [*CreateAggrAgreement*][caa]$(S_i)$
                2. [*Broadcast*][mx]$(\mathsf{M}^{Ag})$
                - $\mathsf{B}^c = \mathsf{H}_{S_i[0]}.BlockHash$
                - $\boldsymbol{V} \leftarrow S_i[0].RVotes$
                1. $\mathsf{B}^w$ = [*MakeWinning*][mw]$(\mathsf{B}^c, \boldsymbol{V}[0], \boldsymbol{V}[1])$
   
   2.  $\texttt{if } (\mathsf{M}^{Ag} =$ [*Receive*][mx]$(\mathsf{AggrAgreement}, r)):$
       - $`(\mathsf{H}_{\mathsf{M}^{Ag}}, \_ , \boldsymbol{bs}, \sigma_A) \leftarrow \mathsf{M}^{Ag}`$
       - $`\_, \mathsf{B}^c, r_{\mathsf{M}^{Ag}}, s_{\mathsf{M}^{Ag}} \leftarrow \mathsf{H}_{\mathsf{M}^{Ag}}`$
       1. [VerifyAggregated][vaa]$(\eta_{\mathsf{B}^c}, r_{\mathsf{M}^{Ag}}, s_{\mathsf{M}^{Ag}}, \boldsymbol{bs}, \sigma_{\boldsymbol{bs}})$
       2. [*Propagate*][mx]$(\mathsf{M}^{Ag})$
       3. $\mathsf{B}^w =$ [*MakeWinning*][mw]$(\mathsf{M}^{Ag}.Agreement)$


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
[mh]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#message-header
[mx]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#message-exchange
<!-- Sortition -->
[bs]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#bitset
[cc]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#countcredits
[sc]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#subcommittee
<!-- Reduction -->
[sv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md#stepvotes
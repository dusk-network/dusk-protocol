# Ratification
When the two [Reduction][red] votes reach a quorum, an [Agreement][amsg] message is produced, containing a [Certificate][cert] with the aggregated votes of the two quorum committees.
Since the certificate proves a candidate reached a quorum, this message is sufficient to accept the voted candidate into the local chain.

The *Ratification* task manages [Agreement][amsg] messages produced by the node or received from the network. In particular, when such a message is received (or produced), the corresponding candidate block is accepted ([*MakeWinning*][mw]).

### ToC
  - [Overview](#overview)
  - [Ratification Procedure](#ratification-procedure)
    - [VerifyAgreement](#verifyagreement)

## Overview
When a node generates or receives an $\mathsf{Agreement}$ message, it adds the included $\mathsf{Certificate}$ to the candidate block and make it a *winning block* for the corresponding round and iteration. 

If the $\mathsf{Agreement}$ message was received from the network, the aggregated votes ([StepVotes][sv]) are verified against the generator and committee members of the corresponding round and iteration. Both the validity of the votes and the quorum quota is verified.

The winning block will then be accepted as the new tip of the local chain.

## Ratification Procedure
The Ratification task manages $\mathsf{Agreement}$ messages for a specific round. If the message was received from the network, it is first verified ([VerifyAgreement][va]) and then propagated.
The corresponding candidate block is then marked as the winning block of the round. 

***Parameters***
- [Consensus Parameters][cparams]

***Algorithm***
1. Loop:
   1. If an $\mathsf{Agreement}$ message $\mathsf{M}^A$ is received for the current round:
      1. Verify $\mathsf{M}^A$ is valid
      2. If valid:
         1. Propagate $\mathsf{M}^A$
         2. Fetch corresponding candidate $\mathsf{B}^c$
         3. Create winning block $\mathsf{B}^w$ with $\mathsf{M}^A.Certificate$

***Procedure***

$\boldsymbol{\textit{Ratification}}( ) :$
1. $\texttt{loop}$:   
   1.  $\texttt{if } (\mathsf{M}^A =$ [*Receive*][mx]$(\mathsf{Agreement}, r)):$
       -  $\texttt{set}:$
          -  $b = \mathsf{H}_{\mathsf{M}^A}.BlockHash$
          -  $\mathsf{C} = \mathsf{M}^A.Certificate$
       1. $isValid$ = [*VerifyAgreement*][va]$(\mathsf{M}^A)$
       2. $\texttt{if } (isValid = true)$:
          1. [*Propagate*][mx]$(\mathsf{M}^A)$
          <!-- TODO: define candidate pool and functions -->
          2. $\mathsf{B}^c = \texttt{fetch}(b)$
          3. [*MakeWinning*][mw]$(\mathsf{B}^c, \mathsf{M}^A.Certificate)$


### VerifyAgreement
*VerifyAgreement* verifies the validity of an $\mathsf{Agreement}$ message.

***Parameters***
- $\mathsf{M}$: $\mathsf{Agreement}$ message

***Algorithm***
1. Verify message signature <!-- TODO: remove this -->
2. Verify Certificate $\mathsf{C}$ ([*VerifyCertificate*][vc])
3. If $\mathsf{C}$ is valid, output true
4. Otherwise, output false

***Procedure***
<!-- TODO: get blockhash from M and retrieve candidate block, if known -->
$VerifyAgreement(\mathsf{M})$
- $`\mathsf{H_M}, \sigma_{\mathsf{M}}, \mathsf{C} \leftarrow \mathsf{M}`$
- $`\_, r_{\mathsf{M}}, s_{\mathsf{M}}, \mathsf{B}^c \leftarrow \mathsf{H_M}`$

1. $isValid = Verify_{BLS}(\mathsf{H_M}, \sigma_{\mathsf{M}}, pk_{\mathsf{M}})$
   1. $\texttt{if } (isValid = false) : \texttt{output } false$
2. $isValid =$ [*VerifyCertificate*][vc]$(\mathsf{C}, \eta_{\mathsf{B}^c}, r_{\mathsf{M}}, s_{\mathsf{M}})$
   1. $\texttt{if } (isValid = false) : \texttt{output } false$
3. $\texttt{output } true$

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md -->
[rat]: #ratification-procedure
[mw]:  #makewinning
[va]:  #verifyagreement

<!-- Consensus -->
[cparams]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#parameters
<!-- Reduction -->
[red]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md 
[sv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md#stepvotes
<!-- Messages -->
[mx]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-exchange
[amsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#agreement-message
<!-- Chain Management -->
[cert]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#certificate
[vc]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#verifycertificate
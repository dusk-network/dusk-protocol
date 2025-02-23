 # Proposal
*Proposal* is the first step in an [*SA iteration*][sai]. In this step, a randomly-extracted provisioner is appointed to generate a new *candidate* block to add to the ledger. In the same step, other provisioners wait for the candidate block produced by the generator.

**ToC**
  - [Overview](#overview)
  - [Procedures](#procedures)
    - [*ProposalStep*](#proposalstep)
    - [*GenerateBlock*](#generateblock)
    - [*SelectTransactions*](#selecttransactions)
    - [*FetchFaults*](#fetchfaults)

## Overview
In the Proposal step, each provisioner node first executes the [*Deterministic Sortition*][ds] algorithm to extract the *block generator*. If the node is selected, it creates a new *candidate block*, and broadcasts it using a [`Candidate`][cmsg] message.
In this step, all other nodes wait to receive the candidate block until the step timeout expires. 
If it was generated or received from the network, the step returns the candidate block; otherwise, it returns $NIL$. The step output will then be passed on as input to the [*Validation*][val] step, where a committee of provisioners will verify its validity and vote accordingly.

<p><br></p>

## Procedures

### *ProposalStep*
This procedure takes in input the round $R$ and the iteration $I$, and outputs the *candidate block* $\mathsf{B}^c$, if it was generated or received, or $NIL$ otherwise.
It is called by [*SAIteration*][sai], which will pass the result to [*ValidationStep*][val].

In the procedure, the node first extracts the *generator* $\mathcal{G}$ with [*ExtractGenerator*][eg]. If the node is extracted as $\mathcal{G}$, it generates the candidate block $\mathsf{B}^c$ and broadcasts it. Otherwise, it waits the Proposal timeout $\tau_{Prop}$ to receive the candidate block from the network. If $\mathsf{B}^c$ is received, it outputs it, otherwise, it outputs $NIL$.

We impose a minimum block time of 10 seconds. This is enforced by having nodes wait until at least 10 seconds have passed since the previous block. This is further verified in the Candidate header verification.

**Parameters** 
- [SA Environment][cenv]
- $R$: round number
- $I$: iteration number

**Algorithm**
1. Extract the block generator $\mathcal{G}$ ([*ExtractGenerator*][eg])
2. If this node's provisioner is $\mathcal{G}$:
   1. Generate candidate block $\mathsf{B}^c$
   2. Create $\mathsf{Candidate}$ message $\mathsf{M}$ containing $\mathsf{B}^c$
   3. Broadcast $\mathsf{M}$
   4. Output $\mathsf{B}^c$
3. Otherwise:
   1. If current time is lower than $Tip$'s timestamp + 10 seconds
      1. Wait until $Tip$'s timestamp + 10 seconds
   2. Start Proposal timeout $\tau_{Prop}$
   3. While timeout has not expired:
      1. If a $\mathsf{Candidate}$ message $\mathsf{M}$ is received for round $R$ and iteration $I$:
         1. If $\mathsf{M}$'s signature is valid
         2. and $\mathsf{M}.PrevHash$ is $Tip$
         3. and $\mathsf{M}$'s signer is $\mathcal{G}$
            1. Propagate $\mathsf{M}$
            2. Store elapsed time
            3. Output $\mathsf{M}$'s block ($\mathsf{B}^c_\mathsf{M}$)
   4. If timeout expired
      1. Increase Proposal timeout
      2. Output $NIL$

**Procedure**

$\textit{ProposalStep}(R, I)$:
1. $\mathcal{G} =$ [*ExtractGenerator*][eg]$(R,I)$
2. $\texttt{if } (pk_\mathcal{N} = \mathcal{G}):$
   1. $\mathsf{B}^c =$ [*GenerateBlock*][gb]$(R,I, Tip)$
   2. $\mathsf{M} =$ [*CMsg*][nmsg]$(\mathsf{Candidate}, \mathsf{B}^c)$
      | Field       | Value               | 
      |-------------|---------------------|
      | $PrevHash$  | $\eta_{Tip}$        |
      | $Round$     | $R$                 |
      | $Iteration$ | $I$                 |
      | $Candidate$ | $\mathsf{B}^c$      |
      | $Signer$    | $pk_\mathcal{N}$    |
      | $Signature$ | $\sigma_\mathsf{M}$ |

   3. [*Broadcast*][broad]$(\mathsf{M})$
   4. $\texttt{output } \mathsf{B}^c$

3. $\texttt{else}:$
   1. $\texttt{if }(\tau_{Now} < Tip.Timestamp+MinBlockTime)$
      1. $\texttt{sleep}(Tip.Timestamp+MinBlockTime - \tau_{Now})$
   2. $\tau_{Start} = \tau_{Now}$
   3. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Prop}) \texttt{ and } (I \lt EmergencyMode):$
      1. $\mathsf{M^C} =$ [*Receive*][recv]$(\mathsf{Candidate},R,I)$
         $\texttt{if } (\mathsf{M^C} \ne NIL):$
         - $`\mathsf{CI}, \mathsf{B}^c, \mathsf{SI} \leftarrow \mathsf{M^C}`$
         - $`\eta_{\mathsf{B}^p}, \_, \_, \leftarrow \mathsf{CI}`$
         - $`pk_\mathsf{M}, \sigma_\mathsf{M} \leftarrow \mathsf{SI}`$
         1. $`\texttt{if }(\text{ }$ [*VerifyMessage*][sigs]$(\mathsf{M^C}) = true \text{ })`$
         2. $\texttt{and }(\eta_{\mathsf{B}^p} = \eta_{Tip})$
         3. $`\texttt{and }(\text{ } pk_\mathsf{M} = \mathcal{G} \text{ }):`$
            1. [*Propagate*][propm]$(\mathsf{M^C})$
            2. [*StoreElapsedTime*][set]$(Proposal, \tau_{Now}-\tau_{Start})$
            3. $\texttt{output } \mathsf{B}^c$
   4. $\texttt{if } (\tau_{Now} > \tau_{Start}+\tau_{Prop}):$
      1. [*IncreaseTimeout*][it]$(Proposal)$
      2. $\texttt{output } NIL$

<p><br></p>

### *GenerateBlock*
This procedure creates a candidate block for round $R$ and iteration $I$ by selecting a set of transactions from the Mempool and executing them to obtain the new state $S$.
It is called by [*ProposalStep*][props], which will broadcast the returned block in a `Candidate` message.

**Parameters**
- $R$: round number
- $I$: iteration number
- $\mathsf{B}^p$: previous block (the current $Tip$)

**Algorithm**
1. Fetch transactions $\boldsymbol{txs}$ from Mempool
2. Fetch faults from DB
3. Execute state transition and get new state $SystemState_R$
4. Compute transaction Merkle tree root
5. Compute fault Merkle tree root
6. Set new $Seed$ by signing the previous one $Seed_{\mathsf{B}^p}$
7. Set block's timestamp
8. Create block header $`\mathsf{H}_{\mathsf{B}^c}`$
9. Create candidate block $\mathsf{B}^c$
10. Output $\mathsf{B}^c$

**Procedure**

$\textit{GenerateBlock}(R,I, \mathsf{B}^p)$
1. $\boldsymbol{txs} =$ [*SelectTransactions*][st]$()$
2. $\boldsymbol{faults} =$ [*FetchFaults*][ff]$()$
3. $SystemState_{\mathsf{B}^c} =$ [*ExecuteStateTransition*][est]$`(\mathsf{B}^p.State, \boldsymbol{txs}, \boldsymbol{faults}, BlockGas, pk_\mathcal{N})`$
4. $`TxRoot_{\mathsf{B}^c} = MerkleTree(\boldsymbol{txs}).Root`$
5. $`FaultRoot_{\mathsf{B}^c} = MerkleTree(\boldsymbol{faults}).Root`$
6. $`Seed_{\mathsf{B}^c} = Sign_{BLS}(sk_\mathcal{N}, Seed_{\mathsf{B}^p})`$
7. $\tau_{\mathsf{B}^c} = Max(\tau_{now}, \mathsf{B}^p.Timestamp+10)$
8. $`\mathsf{H}_{\mathsf{B}^c} = \{Version,R,\tau_{\mathsf{B}^c},BlockGas,I,\eta_{\mathsf{B}_{\mathsf{B}^p}},Seed_{\mathsf{B}^c},pk_\mathcal{N},$
   $TxRoot_R,SystemState_R,\mathsf{B}_{R-1}.Attestation, \boldsymbol{FailedIterations}\}`$
    | Field                  | Value                              | 
    |------------------------|------------------------------------|
    | $Version$              | $V$                                |
    | $Height$               | $R$                                |
    | $Timestamp$            | $\tau_{\mathsf{B}^c}$              |
    | $GasLimit$             | $BlockGas$                         |
    | $Iteration$            | $I$                                |
    | $PreviousBlock$        | $\eta_{\mathsf{B}_{\mathsf{B}^p}}$ |
    | $Seed$                 | $Seed_{\mathsf{B}^c}$              |
    | $Generator$            | $pk_\mathcal{N}$                   |
    | $TxRoot$               | $TxRoot_{\mathsf{B}^c}$            |
    | $FaultRoot$            | $FaultRoot_{\mathsf{B}^c}$         |
    | $State$                | $SystemState_{\mathsf{B}^c}$       |
    | $PrevBlockCertificate$ | $\mathsf{B}^p.Attestation$         | 
    | $FailedIterations$     | $\boldsymbol{FailedIterations}$    |
    
9. $`\mathsf{B}^c = \{\mathsf{H}, \boldsymbol{txs}, \boldsymbol{faults}\}`$
    | Field          | Value                       | 
    |----------------|-----------------------------|
    | $Header$       | $\mathsf{H}_{\mathsf{B}^c}$ |
    | $Transactions$ | $\boldsymbol{txs}$          |
    | $Faults$       | $\boldsymbol{faults}$       |

10. $\texttt{output } \mathsf{B}^c$

<p><br></p>

### *SelectTransactions*
This procedure selects a set of transactions from the Mempool to be included in a new block.

The criteria used for the selection is arbitrary and is left to the Block Generator.
Typically, the Generator's strategy will aim at maximizing profits by selecting transactions paying higher gas price.
In this respect, it can be assumed that transactions paying higher gas prices will be prioritized by most block generators, and will then be included in the blockchain earlier.

A maximum of $MaxTransactions$ (see [Global Parameters][cenv]) items are returned.

### *FetchFaults*
This procedure returns all known faults which have not yet been included in any block.

A maximum of $MaxFaults$ (see [Global Parameters][cenv]) items are returned.
Priority is given to older faults.


<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md -->
[prop]:  #proposal
[props]: #proposalstep
[gb]:    #generateblock
[st]:    #selecttransactions
[ff]:    #fetchfaults

<!-- Basics -->
[eg]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#ExtractGenerator
[gsn]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetStepNum

<!-- Protocol -->
[cenv]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#environment
[set]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#storeelapsedtime
[it]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#increasetimeout
[sai]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#saiteration

[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md

[ds]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md
[dsp]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md#deterministic-sortition-ds

<!-- Messages -->
[sigs]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#signatures
[cmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#candidate
[nmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#cmsg
[broad]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#broadcast
[recv]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#receive
[propm]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#propagate

<!-- TODO: Add ExecuteStateTransition -->
[est]:  https://github.com/dusk-network/dusk-protocol/tree/main/

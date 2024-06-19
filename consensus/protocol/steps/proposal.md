 # Proposal
*Proposal* is the first step in an [*SA iteration*][sai]. In this step, a randomly-extracted provisioner is appointed to generate a new *candidate* block to add to the ledger. In the same step, other provisioners wait for the candidate block produced by the generator.

**ToC**
  - [Overview](#overview)
  - [Procedures](#procedures)
    - [*ProposalStep*](#proposalstep)
    - [*GenerateBlock*](#generateblock)
    - [*SelectTransactions*](#selecttransactions)

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

**Parameters** 
- [SA Environment][env]
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
   1. Start Proposal timeout $\tau_{Prop}$
   2. While timeout has not expired:
      1. If a $\mathsf{Candidate}$ message $\mathsf{M}$ is received for round $R$ and iteration $I$:
         1. If $\mathsf{M}$'s signature is valid
         2. and $\mathsf{M}.PrevHash$ is $Tip$
         3. and $\mathsf{M}$'s signer is $\mathcal{G}$
            1. Propagate $\mathsf{M}$
            2. Store elapsed time
            3. Output $\mathsf{M}$'s block ($\mathsf{B}^c_\mathsf{M}$)
   3. If timeout expired
      1. Increase Proposal timeout
      2. Output $NIL$

**Procedure**

$\textit{ProposalStep}(R, I)$:
1. $\mathcal{G} =$ [*ExtractGenerator*][eg]$(R,I)$
2. $\texttt{if } (pk_\mathcal{N} = \mathcal{G}):$
   1. $\mathsf{B}^c =$ [*GenerateBlock*][gb]$(R,I)$
   2. $\mathsf{M} =$ [*Msg*][msg]$(\mathsf{Candidate}, \mathsf{B}^c)$
      | Field       | Value               | 
      |-------------|---------------------|
      | $PrevHash$  | $\eta_{Tip}$        |
      | $Round$     | $R$                 |
      | $Iteration$ | $I$                 |
      | $Candidate$ | $\mathsf{B}^c$      |
      | $Signer$    | $pk_\mathcal{N}$    |
      | $Signature$ | $\sigma_\mathsf{M}$ |

   3. [*Broadcast*][mx]$(\mathsf{M})$
   4. $\texttt{output } \mathsf{B}^c$
3. $\texttt{else}:$
   1. $\tau_{Start} = \tau_{Now}$
   2. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Prop}) \texttt{ and } (I \lt EmergencyMode):$
      1. $\mathsf{M^C} =$ [*Receive*][mx]$(\mathsf{Candidate},R,I)$
         $\texttt{if } (\mathsf{M^C} \ne NIL):$
         - $`\mathsf{CI}, \mathsf{B}^c, \mathsf{SI} \leftarrow \mathsf{M^C}`$
         - $`\eta_{\mathsf{B}^p}, \_, \_, \leftarrow \mathsf{CI}`$
         - $`pk_\mathsf{M}, \sigma_\mathsf{M} \leftarrow \mathsf{SI}`$
         1. $`\texttt{if }(\text{ }$ [*VerifyMessage*][ms]$(\mathsf{M^C}) = true \text{ })`$
         2. $\texttt{and }(\eta_{\mathsf{B}^p} = \eta_{Tip})$
         3. $`\texttt{and }(\text{ } pk_\mathsf{M} = \mathcal{G} \text{ }):`$
            1. [*Propagate*][mx]$(\mathsf{M^C})$
            2. [*StoreElapsedTime*][set]$(Proposal, \tau_{Now}-\tau_{Start})$
            3. $\texttt{output } \mathsf{B}^c$
   3. $\texttt{if } (\tau_{Now} > \tau_{Start}+\tau_{Prop}):$
      1. [*IncreaseTimeout*][it]$(Proposal)$
      2. $\texttt{output } NIL$

<p><br></p>

### *GenerateBlock*
This procedure creates a candidate block for round $R$ and iteration $I$ by selecting a set of transactions from the Mempool and executing them to obtain the new state $S$.
It is called by [*ProposalStep*][props], which will broadcast the returned block in a `Candidate` message.

**Parameters**
- [SA Environment][env]
- $R$: round number
- $I$: iteration number

**Algorithm**
1. Fetch transactions $\boldsymbol{txs}$ from Mempool
2. Execute $\boldsymbol{txs}$ and get new state $State_R$
3. Compute transaction Merkle tree root $TxRoot_R$
4. Set new $Seed_R$ by signing the previous one $Seed_{R-1}$
5. Create block header $`\mathsf{H}_{\mathsf{B}^c_{R,I}}`$
6. Create candidate block $\mathsf{B}^c$
7. Output $\mathsf{B}^c$

**Procedure**

$\textit{GenerateBlock}(R,I)$
1. $`\boldsymbol{txs} = [tx_1, \dots, tx_n] = `$ [*SelectTransactions*][st]$()$
2. $SystemState_R =$ [*ExecuteStateTransition*][est]$`(State_{R-1}, \boldsymbol{txs}, BlockGas,pk_\mathcal{N})`$
3. $`TxRoot_R = MerkleTree(\boldsymbol{txs}).Root`$
4. $`Seed_R = Sign_{BLS}(sk_\mathcal{N}, Seed_{R-1})`$
5. $`\mathsf{H}_{\mathsf{B}^c_{R,I}} = (V,R,\tau_{now},BlockGas,I,\eta_{\mathsf{B}_{R-1}},Seed_R,pk_\mathcal{N},$
   $TxRoot_R,SystemState_R,\mathsf{B}_{R-1}.Attestation, \boldsymbol{FailedAttestations})`$
    | Field                  | Value                              | 
    |------------------------|------------------------------------|
    | $Version$              | $V$                                |
    | $Height$               | $R$                                |
    | $Timestamp$            | $\tau_{now}$                       |
    | $GasLimit$             | $BlockGas$                         |
    | $Iteration$            | $I$                                |
    | $PreviousBlock$        | $\eta_{\mathsf{B}_{R-1}}$          |
    | $Seed$                 | $Seed_R$                           |
    | $Generator$            | $pk_\mathcal{N}$                   |
    | $TxRoot$               | $TxRoot_R$                         |
    | $State$                | $SystemState_R$                    |
    | $PrevBlockCertificate$ | $\mathsf{B}_{R-1}.Attestation$     | 
    | $FailedIterations$     | $\boldsymbol{FailedAttestations}$  |
    
6. $`\mathsf{B}^c_{R,I} = (\mathsf{H}, \boldsymbol{tx})`$
    | Field          | Value                       | 
    |----------------|-----------------------------|
    | $Header$       | $\mathsf{H}_{\mathsf{B}^c}$ |
    | $Transactions$ | $\boldsymbol{txs}$          |
7. $\texttt{output } \mathsf{B}^c_{R,I}$

<p><br></p>

### *SelectTransactions*
This procedure selects a set of transactions from the Mempool to be included in a new block.

The criteria used for the selection is arbitrary and is left to the Block Generator.
Typically, the Generator's strategy will aim at maximizing profits by selecting transactions paying higher gas price.
In this respect, it can be assumed that transactions paying higher gas prices will be prioritized by most block generators, and will then be included in the blockchain earlier.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md -->
[prop]:  #proposal
[props]: #proposalstep
[gb]:    #generateblock
[st]:    #selecttransactions

<!-- Basics -->
[eg]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#ExtractGenerator
[gsn]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetStepNum

<!-- Protocol -->
[env]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#environment
[set]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#storeelapsedtime
[it]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#increasetimeout
[sai]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#saiteration

[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md

[ds]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md
[dsp]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md#deterministic-sortition-ds

<!-- Messages -->
[ms]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#signatures
[msg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#msg
[mx]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#procedures
[cmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#candidate

<!-- TODO: Add ExecuteTransactions -->
[est]:  https://github.com/dusk-network/dusk-protocol/tree/main/

 # Proposal
*Proposal* is the first step in an [*SA iteration*][sai]. In this step, a randomly-extracted provisioner is appointed to generate a new *candidate* block to add to the ledger. In the same step, other provisioners wait for the candidate block produced by the generator.

#### ToC
- [Overview](#overview)
- [Procedures](#procedures)
  - [*ProposalStep*](#proposalstep)
  - [*GenerateBlock*](#generateblock)
  - [*SelectTransactions*](#selecttransactions)


## Overview
In the Proposal step, each provisioner node first executes the [*Deterministic Sortition*][ds] algorithm to extract the *block generator*. If the node is selected, it creates a new *candidate block*, and broadcasts it using a $\mathsf{Candidate}$ message (see [`Candidate`][cmsg]).
In this step, all other nodes wait to receive the candidate block until the step timeout expires. 
If it was generated or received from the network, the step returns the candidate block; otherwise, it returns $NIL$. The step output will then be passed on as input to the [*Validation*][val] step, where a committee of provisioners will verify its validity and vote accordingly.

<p><br></p>

## Procedures

### *ProposalStep*
*ProposalStep* takes in input the round $R$ and the iteration $I$, and outputs the *candidate block* $\mathsf{B}^c$, if it was generated or received, or $NIL$ otherwise.
It is called by [*SAIteration*][sai], which will pass the result to [*ValidationStep*][val].

In the procedure, the node first extracts the *generator* $\mathcal{G}$ using [*DS*][dsa]. If the node's extracted, it generates the candidate $\mathsf{B}^c$ and broadcasts it. Otherwise, it waits $\tau_{Proposal}$ (the Proposal timeout) to receive the candidate block from the network. If the block is received, it outputs it, otherwise, it outputs $NIL$.

***Parameters*** 
- [SA Environment][env]
- $R$: round number
- $I$: iteration number

***Algorithm***
1. Extract the block generator ($\mathcal{G}$) [*DS*][dsa]$(R,S,1)$
2. If this node's provisioner is the block generator:
   1. Generate candidate block $\mathsf{B}^c$
   2. Create $\mathsf{Candidate}$ message $\mathsf{M}$ containing $\mathsf{B}^c$
   3. Broadcast $\mathsf{M}$
   4. Output $\mathsf{B}^c$
3. Otherwise:
   1. Start Proposal timeout $\tau_{Proposal}$
   2. While timeout has not expired:
      1. If a $\mathsf{Candidate}$ message $\mathsf{M}$ is received for round $R$ and iteration $I$:
         1. If $\mathsf{M}$'s signature is valid
         2. and $\mathsf{M}$'s signer is $\mathcal{G}$
         3. and $\mathsf{M}$'s $BlockHash$ is $Candidate$'s hash
            1. Propagate $\mathsf{M}$
            2. Output $\mathsf{M}$'s block ($\mathsf{B}^c_\mathsf{M}$)
   3. If timeout expired
      1. Increase Proposal timeout
      2. Output $NIL$

***Procedure***

$\textit{Proposal}(R, I)$:
1. $\texttt{set}$:
   - $s = I \times 3$
2. $pk_{\mathcal{G}} =$ [*DS*][dsa]$(R,s,1)$
3. $\texttt{if } (pk_\mathcal{N} = pk_{\mathcal{G}}):$
   1. $\mathsf{B}^c =$ [*GenerateBlock*][gb]$(R,I)$
   2. $\mathsf{M} =$ [*Msg*][msg]$(\mathsf{Candidate}, \mathsf{B}^c)$
      | Field       | Value                     | 
      |-------------|---------------------------|
      | $Header$    | $\mathsf{H}_\mathsf{M}$   |
      | $Candidate$ | $\mathsf{B}^c$            |
   3. [*Broadcast*][mx]$(\mathsf{M})$
   4. $\texttt{output } \mathsf{B}^c$
4. $\texttt{else}:$
   1. $\tau_{Start} = \tau_{Now}$
   2. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Proposal}):$
      1. $\mathsf{M} = $ [*Receive*][mx]$(\mathsf{Candidate},R,I)$
         $\texttt{if } (\mathsf{M} \ne NIL):$
         - $`(\mathsf{H}_\mathsf{M}, \mathsf{B}_\mathsf{M}) \leftarrow \mathsf{M}`$
         - $`(\_, pk_\mathsf{M}, \sigma_\mathsf{M}) \leftarrow \mathsf{H}_\mathsf{M}`$
         1. $`\texttt{if }(\text{ }$ [*VerifyMessage*][ms]$(\mathsf{M}) = true \text{ })`$
         2. $`\texttt{and }(\text{ } pk_\mathsf{M} = pk_{\mathcal{G}} \text{ })`$
         3. $`\texttt{and } (\text{ }\eta_\mathsf{M} = \eta_{\mathsf{B}_\mathsf{M}} \text{ }):`$
            1. [*Propagate*][mx]$(\mathsf{M})$
            2. $\texttt{output } \mathsf{B}_\mathsf{M}$
   3. $\texttt{if } (\tau_{Now} > \tau_{Start}+\tau_{Proposal}):$
      1. [*IncreaseTimeout*][it]$(\tau_{Proposal})$
      2. $\texttt{output } NIL$

<p><br></p>

### *GenerateBlock*
*GenerateBlock* creates a candidate block for round $R$ and iteration $I$ by selecting a set of transactions from the Mempool and executing them to obtain the new state $S$.
It is called by [*ProposalStep*][ps], which will broadcast the returned block in a $\mathsf{Candidate}$ message.

***Parameters***
- [SA Environment][env]
- $R$: round number
- $I$: iteration number

***Algorithm***
1. Fetch transactions $\boldsymbol{txs}$ from Mempool
2. Execute $\boldsymbol{txs}$ and get new state $State_R$
3. Compute transaction Merkle tree root $TxRoot_R$
4. Set new $Seed_R$ by signing the previous one $Seed_{R-1}$
5. Create block header $\mathsf{H}_{\mathsf{B}^c_{R,I}}$
6. Create candidate block $\mathsf{B}^c$
7. Output $\mathsf{B}^c$

***Procedure***

$GenerateBlock(R,I)$
1. $`\boldsymbol{txs} = [tx_1, \dots, tx_n] = `$ [*SelectTransactions*][st]$()$
2. $State_R =$ [*ExecuteStateTransition*][xt]$`(State_{R-1}, \boldsymbol{txs}, BlockGas,pk_\mathcal{N})`$
3. $`TxRoot_R = MerkleTree(\boldsymbol{txs}).Root`$
4. $`Seed_R = Sign_{BLS}(sk_\mathcal{N}, Seed_{R-1})`$
5. $`\mathsf{H}_{\mathsf{B}^c_{R,I}} = (V,R,\tau_{now},BlockGas,I,\eta_{\mathsf{B}_{R-1}},Seed_R,pk_\mathcal{N},$
   $TxRoot_R,State_R,\mathsf{B}_{R-1}.Certificate, \boldsymbol{FailedCertificates})`$
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
    | $State$                | $State_R$                          |
    | $PrevBlockCertificate$ | $\mathsf{B}_{r-1}.Certificate$     | 
    | $FailedIterations$     | $\boldsymbol{FailedCertificates}$  |
    
6. $`\mathsf{B}^c_{r,i} = (\mathsf{H}, \boldsymbol{tx})`$
    | Field          | Value                       | 
    |----------------|-----------------------------|
    | $Header$       | $\mathsf{H}_{\mathsf{B}^c}$ |
    | $Transactions$ | $\boldsymbol{txs}$           |
7. $\texttt{output } \mathsf{B}^c_{r,i}$

<p><br></p>

### *SelectTransactions*
*SelectTransactions* selects a set of transactions from the Mempool to be included in a new block.

The criteria used for the selection is arbitrary and is left to the Block Generator.
Typically, the Generator's strategy will aim at maximizing profits by selecting transactions paying higher gas price.
In this respect, it can be assumed that transactions paying higher gas prices will be prioritized by most block generators, and will then be included in the blockchain earlier.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal/README.md -->
[prop]: #proposal
[ps]: #proposalstep
[gb]: #generateblock
[st]: #selecttransactions

<!-- Consensus -->
[env]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#environment
[it]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#increasetimeout
[sai]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#saiteration
<!-- Sortition -->
[ds]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/
[dsa]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#deterministic-sortition-ds
<!-- Validation -->
[val]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md

<!-- TODO: Add ExecuteTransactions -->
[xt]: https://github.com/dusk-network/dusk-protocol/tree/main/

<!-- Messages -->
[msg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-creation
[mx]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#procedures
[cmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#candidate-message
[ms]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#signatures
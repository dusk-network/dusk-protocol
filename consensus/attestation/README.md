 # Attestation Phase
*Attestation* is the first phase in an [*SA iteration*][cit].
In this phase, a selected provisioner is appointed to generate a new block. All other provisioners in this phase will wait a certain time to receive this block. 

## Phase Overview
Each provisioner node first executes the [*Deterministic Sortition*][ds] algorithm to check who's selected as the *block generator*. If the node itself is selected, it creates a new *candidate block*, and broadcasts it to the network via a $\mathsf{NewBlock}$ message.
Otherwise, the node waits a certain timeout to receive the candidate block from the network. If such a block is received and it is signed by the extracted block generator, it propagates the message and moves to the [*Reduction*][red] phase, where it will vote on the block validity. All other blocks are discarded.

If the timeout expires, it moves to Reduction with an empty candidate block ($NIL$).


### NewBlock Message
The $\mathsf{NewBlock}$ message is used by a block generator to broadcast a candidate block.

The message has the following structure:

| Field       | Type                  | Size      | Description           |
|-------------|-----------------------|-----------|-----------------------|
| $Header$    | [*MessageHeader*][mh] | 137 bytes | Message header        |
| $PrevHash$  | SHA3 Hash             | 256 bits  | Previous block's hash |
| $Candidate$ | [*Block*][b]          |           | Candidate block       |
| $Signature$ | BLS Signature         | 48 bytes  | Message signature     |

The $\mathsf{NewBlock}$ message has a variable size of 217 bytes plus the block size.

## Attestation Algorithm
***Parameters*** 
- [Consensus Parameters][cp]
- $Round$: round number
- $Iteration$: iteration number

***Algorithm***
1. Extract the block generator ($\mathcal{G}$) [*DS*][dsa]$(R,S,1)$
2. If this node's provisioner is the block generator:
   1. Generate candidate block $\mathsf{B}^c$
   2. Create $\mathsf{NewBlock}$ message $\mathsf{M}^B$ containing $\mathsf{B}^c$
   3. Broadcast $\mathsf{M}^B$
   4. Output candidate block
3. Otherwise:
   1. Start Attestation timeout
   2. While timeout is not expired:
      1. If a $\mathsf{NewBlock}$ message $\mathsf{M}^B$ is received for this round and step:
         1. If $\mathsf{M}^B$'s signature is valid
         2. and $\mathsf{M}^B$'s signer is $\mathcal{G}$
         3. and $\mathsf{M}^B$'s $BlockHash$ corresponds to $Candidate$
            1. Propagate $\mathsf{M}^B$
            2. Output $\mathsf{M}^B$'s block ($\mathsf{B}^\mathsf{M}$)
   3. If timeout expired
      1. Increase Attestation timeout
      2. Output $NIL$

***Procedure***

$Attestation(Round, Iteration)$:
1. $\texttt{set}$:
   - $r = Round$
   - $s = (Iteration-1) \times 3 + 1$
2. $pk_{\mathcal{G}} =$ [*DS*][dsa]$(r,s,1)$
3. $\texttt{if } (pk_\mathcal{N} == pk_{\mathcal{G}}):$
   1. $\mathsf{B}^c =$ [*GenerateBlock*](#generateblock)$()$
   2. $\mathsf{M}^B =$ [*Msg*][msg]$(\mathsf{NewBlock}, \eta_{\mathsf{B}_{r-1}}, \mathsf{B}^c)$
      | Field       | Value                     | 
      |-------------|---------------------------|
      | $Header$    | $\mathsf{H}_\mathsf{M}$   |
      | $PrevHash$  | $\eta_{\mathsf{B}_{r-1}}$ |
      | $Candidate$ | $\mathsf{B}^c$            |
      | $Signature$ | $\sigma_\mathsf{M}$       |
   3. $Broadcast(\mathsf{M}^B)$
   4. $\texttt{output } \mathsf{B}^c$
4. $\texttt{else}:$
   1. $\tau_{Start} = \tau_{Now}$
   2. $\texttt{while } (\tau_{now} \le \tau_{Start}+\tau_{Attestation}):$
      1. $\texttt{if } (\mathsf{M} {=} Receive(\mathsf{NewBlock},r,s) \ne NIL)$:
         - $`(\mathsf{H}_\mathsf{M},\_,\mathsf{B}^\mathsf{M},\sigma_\mathsf{M}) \leftarrow \mathsf{M}`$
         - $`\eta_{\mathsf{B}^\mathsf{M}} = Hash_{SHA3}(\mathsf{H}^{\mathsf{B}^\mathsf{M}})`$
         - $`(pk_\mathsf{M},\_,\_,\eta_\mathsf{B}^\mathsf{M}) \leftarrow \mathsf{H}_\mathsf{M}`$
         1. $`\texttt{if }(\text{ } VerifySignature(\mathsf{M}) == true \text{ })`$
         2. $`\texttt{and }(\text{ } pk_\mathsf{M} == pk_{\mathcal{G}} \text{ })`$
         3. $`\texttt{and } (\text{ }\eta_\mathsf{B}^\mathsf{M} == \eta_{\mathsf{B}^\mathsf{M}} \text{ }):`$
            1. $Propagate(\mathsf{M})$
            2. $\texttt{output } \mathsf{B}^\mathsf{M}$
   3. $\texttt{if } (\tau_{Now} > \tau_{Start}+\tau_{Attestation}):$
      1. [*IncreaseTimeout*][it]$(\tau_{Attestation})$
      2. $\texttt{output } NIL$

<p><br></p>

### GenerateBlock

***Algorithm***
1. Fetch transactions from Mempool
2. Execute transactions and get new state hash
3. Compute transaction tree root
4. Set timestamp to current time
5. Compute iteration number
6. Set new $Seed$ by signing the previous one
7. Create header
8. Create candidate block
9. Output candidate block

***Procedure***

$GenerateBlock()$
1. $`\boldsymbol{txs} = [tx_1, \dots, tx_n] = `$ [*SelectTransactions*](#selecttransactions)()
2. $State_r =$ [*ExecuteTransactions*]()$`(\boldsymbol{txs}, BlockGas,pk_\mathcal{N})`$
3. $`TxRoot_r = MerkleTree(\boldsymbol{txs}).Root`$
4. $`i = \lfloor\frac{s}{3}\rfloor`$
5. $`Seed_r = Sign_{BLS}(sk_\mathcal{N}, Seed_{r-1})`$
6. $`\mathsf{H}_{\mathsf{B}^c_{r,i}} = (v,r,\tau_{now},BlockGas,i,\eta_{\mathsf{B}_{r-1}},Seed_r,pk_\mathcal{N},TxRoot_r,State_r)`$
    | Field           | Value                     | 
    |-----------------|---------------------------|
    | $Version$       | $V$                       |
    | $Height$        | $r$                       |
    | $Timestamp$     | $\tau_{now}$              |
    | $GasLimit$      | $BlockGas$                |
    | $Iteration$     | $i$                       |
    | $PreviousBlock$ | $\eta_{\mathsf{B}_{r-1}}$ |
    | $Seed$          | $Seed_r$                  |
    | $Generator$     | $pk_\mathcal{N}$          |
    | $TxRoot$        | $TxRoot_r$                |
    | $State$         | $State_r$                 |

    <!-- | $Header Hash           | string | -->
    <!-- | Certificate           |    ?   | -->
7. $`\mathsf{B}^c_{r,i} = (\mathsf{H}, \boldsymbol{tx})`$
    | Field          | Value                       | 
    |----------------|-----------------------------|
    | $Header$       | $\mathsf{H}_{\mathsf{B}^c}$ |
    | $Transactions$ | $\boldsymbol{tx}$           |
8. $\texttt{output } \mathsf{B}^c_{r,i}$

<p><br></p>

### SelectTransactions
$`SelectTransactions`$ selects a set of transactions from the Mempool to be included in a new block.
The criteria used for the selection is arbitrary and is left to the Block Generator.

Typically, the Generator's strategy will aim at maximizing profits by selecting transactions paying higher gas price.
In this respect, it can be assumed that transactions paying higher gas prices will be prioritized by most block generators, and will then be included in the blockchain earlier.

<!------------------------- LINKS ------------------------->

[cp]: ../README.md#consensus-parameters
[cit]: ../README.md#saiteration
[mh]: ../README.md#message-header
[b]: ../../blockchain/README.md#block-structure
[ds]: ../sortition/
[dsa]: ../sortition/README.md#deterministic-sortition-ds
[msg]: ../README.md#create-message
[red]: ../reduction/README.md
[it]: ../README.md#increasetimeout
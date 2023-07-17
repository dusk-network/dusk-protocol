# Attestation Phase
*Attestation* is the first phase in an [*SA iteration*](../README.md#workflow).
In this phase, a selected Provisioner is appointed to generate a new block. All other provisioners in this phase will wait a certain time to receive this block. 

## Phase Overview
Each provisioner node first executes the [*Deterministic Sortition*](../sortition/) algorithm to check who's selected as the *block generator*. If the node itself is selected, it creates a new *candidate block*, and broadcasts it to the network via a $NewBlock$ message.
Otherwise, the node waits a certain timeout to receive the candidate block from the network. If such a block is received and it is signed by the extracted block generator, it propagates the message and moves to the [*Reduction*](../reduction/) phase, where it will vote on the block validity. All other blocks are discarded.

If the timeout expires, it moves to Reduction with an empty candidate block ($NIL$).


<!-- TODO: Block Generator -->

### Candidate Block
A candidate block is the block generated in the Attestation step by the extracted Provisioner. This is the block on which other provisioners will have to reach an agreement. If an agreement is not reached by the end of the iteration, a new candidate block will be produced and a new iteration will start.

Therefore, for each iteration, only one (valid) candidate block can be produced[^1]. To reflect this, we denote a candidate block with $\mathcal{B}^c_{r,i}$, where $r$ is the consensus round, and $i$ is the consensus iteration.

Note that we simplify this notation to simply $\mathcal{B}^c$ when this does not generate

### NewBlock Message
The $NewBlock$ message is used by a block generator to broadcast a candidate block.

The message has the following structure:

| Field       | Type                  | Size      | Description           |
|-------------|-----------------------|-----------|-----------------------|
| $Header$    | [*MessageHeader*][mh] | 137 bytes | Message header        |
| $PrevHash$  | SHA3 Hash             | 256 bits  | Previous block's hash |
| $Candidate$ | [*Block*][b]          |           | Candidate block       |
| $Signature$ | BLS Signature         | 48 bytes  | Message signature     |

The $NewBlock$ message has a variable size of 217 bytes plus the block size.

## Attestation Algorithm
***Parameters***: 
- [Consensus Parameters](../README.md#parameters)

***Algorithm***:
1. Extract the block generator ($G$) [*DS*][ds]$(R,S,1)$
2. If this node is the block generator:
   1. Generate candidate block $\mathcal{B}^c$ [ [_GenerateBlock_()](#generateblock) ]
   2. Create $NewBlock$ message $\mathcal{M}^B$ containing $\mathcal{B}^c$
   3. Broadcast $\mathcal{M}^B$
   4. Execute first $Reduction$ with $\mathcal{B}^c$
3. Otherwise:
   1. Start Attestation timeout
   2. Loop:
      1. If a $NewBlock$ message $\mathcal{M}^B$ is received for this round and step:
         1. If $\mathcal{M}^B$'s signature is valid
         2. and $\mathcal{M}^B$'s signer is $G$
         3. and $\mathcal{M}^B$'s $BlockHash$ corresponds to $Candidate$
            1. Propagate $\mathcal{M^B}$
            2. Execute first $Reduction$ with $Candidate$
      2. If timeout expired
         1. Execute first $Reduction$ with $NIL$

***Procedure***:

$RunAttestation()$:
1. $pk_{G} = $ [*DS*][ds]$(r,s,1)$
2. $if \text{ } pk_\mathcal{N} == pk_{G}$:
   1. $\mathcal{B}^c =$ [_GenerateBlock_](#generateblock)$()$
   2. $\mathcal{M}^B =$ [*Msg*][msg]$(NewBlock, \eta_{\mathcal{B}_{r-1}}, \mathcal{B}^c)$
      | Field       | Value                      | 
      |-------------|----------------------------|
      | $Header$    | $\mathcal{H}_\mathcal{M}$  |
      | $PrevHash$  | $\eta_{\mathcal{B}_{r-1}}$ |
      | $Candidate$ | $\mathcal{B}^c$            |
      | $Signature$ | $\sigma_\mathcal{M}$       |
   3. $Broadcast(\mathcal{M}^B)$
   4. [*RunReduction*][red]$(\mathcal{B}^c, 1)$
3. $else$:
   1. $\tau_{Start} = \tau_{Now}$
   2. $loop$:  <!-- TODO: change this to while t_now <= t_start + timeout  -->
      1. $if (\mathcal{M} {=} Receive(NewBlock,r,s) \ne NIL)$:
         - $`(\mathcal{H}_\mathcal{M},\_,\mathcal{B}^c,\sigma_\mathcal{M}) \leftarrow \mathcal{M}`$
         - $`\eta_{\mathcal{B}^c} = H_{SHA3}(\mathcal{H}^{\mathcal{B}^c})`$
         - $`(pk_\mathcal{M},\_,\_,\eta_\mathcal{B}^\mathcal{M}) \leftarrow \mathcal{H}_\mathcal{M}`$
         1. $`if \text{ }(\text{ } VerifySignature(\mathcal{M}) == true \text{ })`$
         2. $`and \text{ }(\text{ } pk_\mathcal{M} == pk_{G} \text{ })`$
         3. $`and \text{ } (\text{ }\eta_\mathcal{B}^\mathcal{M} == \eta_{\mathcal{B}^c} \text{ })`$:
            1. $Propagate(\mathcal{M})$
            2. [*RunReduction*][red]$(\mathcal{B}^c, 1)$
      2. $if \text{ } \tau_{Now} > \tau_{Start}+\tau_{Attestation}$
         1. [*RunReduction*][red]$(NIL, 1)$

<p><br></p>

#### GenerateBlock
***Parameters***:
- [Consensus Parameters](../README.md#parameters)

***Algorithm***:
1. Fetch transactions from Mempool
2. Execute transactions and get new state hash
3. Compute transaction tree root
4. Set timestamp to current time
5. Compute iteration number
6. Set new $Seed$ by signing the previous one
7. Create header
8. Create candidate block
9. Output candidate block

***Procedure***:
$GenerateBlock()$
1. $`\boldsymbol{tx} = [tx_1, \dots, tx_n] = `$ [_SelectTransactions_](#selecttransactions)()
2. $State_r =$ [_ExecuteTransactions_](../../vm)$`(State_{r-1}, \boldsymbol{tx})`$
3. $`TxRoot_r = MerkleTree(\boldsymbol{tx}).Root`$
4. $`i = \lfloor\frac{s}{3}\rfloor`$
5. $`Seed_r = Sign_{BLS}(sk_\mathcal{N}, Seed_{r-1})`$
6. $`\mathcal{H}_{\mathcal{B}^c_{r,i}} = (v,r,\tau_{now},Gas^{\mathcal{B}},i,\eta_{\mathcal{B}_{r-1}},Seed_r,pk_\mathcal{N},TxRoot_r,State_r)`$
    | Field           | Value               | 
    |-----------------|---------------------|
    | $Version$       | $V$                 |
    | $Height$        | $r$                 |
    | $Timestamp$     | $\tau_{now}$        |
    | $GasLimit$      | $Gas^{\mathcal{B}}$ |
    | $Iteration$     | $i$                 |
    | $PreviousBlock$ | $\eta_{\mathcal{B}_{r-1}}$ |
    | $Seed$          | $Seed_r$            |
    | $Generator$     | $pk_\mathcal{N}$    |
    | $TxRoot$        | $TxRoot_r$          |
    | $State$         | $State_r$           |

    <!-- | $Header Hash           | string | -->
    <!-- | Certificate           |    ?   | -->
7. $`\mathcal{B}^c_{r,i} = (\mathcal{H}, \boldsymbol{tx})`$
    | Field          | Value                         | 
    |----------------|-------------------------------|
    | $Header$       | $\mathcal{H}_{\mathcal{B}^c}$ |
    | $Transactions$ | $\boldsymbol{tx}$            |
8. $output \text{ } \mathcal{B}^c_{r,i}$

<p><br></p>

#### SelectTransactions
$`SelectTransactions`$ selects a set of transactions from the Mempool to be included in a new block.
The criteria used for the selection is arbitrary and is left to the Block Generator.

Typically, the Generator's strategy will aim at maximizing profits by selecting transactions paying higher gas price.
In this respect, it can be assumed that transactions paying higher gas prices will be prioritized by most block generators, and will then be included in the blockchain earlier.

<!-- TODO: In our implementation:
To ease this process, transactions in the Mempool are ordered by their gas price. -->

<!-- FOOTNOTES -->
[^1]: In principle, a malicious block generator could create two valid candidate blocks. However, this case is automatically handled in the Reduction phase, since provisioners will reach agreement on a specific block hash.

<!--  -->
[mh]: ../README.md#consensus-message-header
[b]: ../../blockchain/README.md#block-structure
[ds]: ../sortition/README.md#deterministic-sortition-ds
[msg]: ../README.md#create-message
[red]: ../reduction/README.md
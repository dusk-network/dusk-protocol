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

Therefore, for each iteration, only one (valid) candidate block can be produced[^1]. To reflect this, we denote a candidate block with $\mathcal{B}_r^i$, where $r$ is the consensus round, and $i$ is the consensus iteration.

### NewBlock Message
The $NewBlock$ message is used by a block generator to broadcast a candidate block.

The message has the following structure:

| Field       | Type                  | Size     | Description           |
|-------------|-----------------------|----------|-----------------------|
| $Header$    | [*MessageHeader*][mh] |          | Message header        |
| $PrevHash$  | SHA3 Hash             | 256 bits | Previous block's hash |
| $Candidate$ | [*Block*][b]          |          | Candidate block       |
| $Signature$ | BLS Signature         | 48 bytes | Message signature     |

## Attestation Algorithm
*Parameters*: 
 - $r$: current round
 - $s$: current step
 - $\tau_{Attestation}$: maximum time for Attestation step (see [SA Parameters](../README.md#parameters))

*Algorithm*:
1. Extract the block generator ($BG$) [*DS*][ds]$(R,S,1)$
2. If this node is the block generator:
   1. Generate candidate block $\mathcal{B}_r^i$ [ [_GenerateBlock_()](#generateblock) ]
   2. Create $NewBlock$ message $\mathcal{M}$ containing $\mathcal{B}_r^i$
   3. Broadcast $\mathcal{M}$
   4. Execute first $Reduction$ with $\mathcal{B}_r^i$
3. Otherwise:
   1. Start Attestation timeout
   2. Loop:
      1. If a $NewBlock$ message $\mathcal{M}$ is received for this round and step:
         1. If $\mathcal{M}$'s signature is valid
         2. and $\mathcal{M}$'s signer is $BG$
         3. and $\mathcal{M}$'s $BlockHash$ corresponds to $Candidate$
            1. Execute first $Reduction$ with $Candidate$
      2. If timeout expired
         1. Execute first $Reduction$ with $NIL$$

*Procedure*:
1. $pk_{BG} = $ [*DS*][ds]$(R,S,1)$
2. $if \text{ } pk_N == pk_{BG}$:
   1. $\mathcal{B}_r^i =$ [_GenerateBlock_](#generateblock)()
   2. $\mathcal{M} =$ [_CreateNBM_](#createnbm)()
   3. $Broadcast(\mathcal{M})$
   4. $Reduction(\mathcal{B}_r^i, 1)$
3. $else$:
   1. $\tau_{Start} = \tau_{Now}$
   2. $loop$:
      1. $if (\text{ } \mathcal{M} = Receive(NewBlock,r,s) \text{ })$:
         - $`(\mathcal{H}_\mathcal{M},\_,\mathcal{B}_\mathcal{M},\sigma_\mathcal{M}) \leftarrow \mathcal{M}`$
         - $`\eta_{\mathcal{B}_\mathcal{M}} = H_{SHA3-256}(\mathcal{B}_\mathcal{M}.Header)`$
         - $`(pk_\mathcal{M},\_,\_,\eta_\mathcal{M}) \leftarrow \mathcal{H}_\mathcal{M}`$
         1. $`if \text{ }(\text{ } Verify_{BLS}(\sigma_\mathcal{M}, pk_\mathcal{M}) == true \text{ })`$
         2. $`and \text{ }(\text{ } pk_\mathcal{M} == pk_{BG} \text{ })`$
         3. $`and \text{ } (\text{ }\eta_\mathcal{M} == \eta_{\mathcal{B}_\mathcal{M}} \text{ })`$:
                1. $Reduction(\mathcal{B}_\mathcal{M}, 1)$
      2. $if \text{ } \tau_{Now} > \tau_{Start}+\tau_{Attestation}$
         1. $Reduction(NIL, 1)$

<p><br></p>

#### GenerateBlock
*Parameters*
- $r$: consensus round
- $s$: consensus step
- $v$: protocol version
- $\eta_{\mathcal{B}_{r-1}}$: previous block's hash

*Algorithm*
1. Fetch transactions from Mempool
2. Execute transactions and get new state hash
3. Compute transaction tree root
4. Set timestamp to current time
5. Compute iteration number
6. Set new $Seed$ by signing the previous one
7. Create header
8. Create candidate block
9. Output candidate block

*Procedure*:
1. $`\boldsymbol{tx} = [tx_1, \dots, tx_n] = `$ [_SelectTransactions_](#selecttransactions)()
2. $State_r =$ [_ExecuteTransactions_](../../vm)$`(State_{r-1}, \boldsymbol{tx})`$
3. $`TxRoot_r = MerkleTree(\boldsymbol{tx}).Root`$
4. $`i = \lfloor\frac{s}{3}\rfloor`$
5. $`Seed_r = Sign_{BLS}(sk_N, Seed_{r-1})`$
6. $`\mathcal{H}_r = (v,r,\tau_{now},Gas^{\mathcal{B}},i,\eta_{\mathcal{B}_{r-1}},Seed_r,pk_N,TxRoot_r,State_r)`$
    | Field           | Value               | 
    |-----------------|---------------------|
    | $Version$       | $v$                 |
    | $Height$        | $r$                 |
    | $Timestamp$     | $\tau_{now}$        |
    | $GasLimit$      | $Gas^{\mathcal{B}}$ |
    | $Iteration$     | $i$                 |
    | $PreviousBlock$ | $\eta_{\mathcal{B}_{r-1}}$ |
    | $Seed$          | $Seed_r$            |
    | $Generator$     | $pk_N$              |
    | $TxRoot$        | $TxRoot_r$          |
    | $State$         | $State_r$           |

    <!-- | $Header Hash           | string | -->
    <!-- | Certificate           |    ?   | -->
7. $`\mathcal{B}_r^i = (\mathcal{H}, \boldsymbol{tx})`$
    | Field         | Value           | 
    |---------------|-----------------|
    | $Header$      | $\mathcal{H}_r$ |
    | $Transaction$ | $\boldsymbol{tx}$     |
8. $output \text{ } \mathcal{B}_r^i$

<p><br></p>

#### SelectTransactions
$`SelectTransactions`$ selects a set of transactions from the Mempool to be included in a new block.
The criteria used for the selection is arbitrary and is left to the Block Generator.

Typically, the Generator's strategy will aim at maximizing profits by selecting transactions paying higher gas price.
In this respect, it can be assumed that transactions paying higher gas prices will be prioritized by most block generators, and will then be included in the blockchain earlier.

<!-- TODO: In our implementation:
To ease this process, transactions in the Mempool are ordered by their gas price. -->

#### CreateNBM
$CreateNBM$ creates a $NewBlock$ message with the new candidate block and the provisioner's signature.

*Parameters*:
  - $r$: consensus round
  - $s$: consensus step
  - $\mathcal{B}_r^i$: candidate block

*Algorithm*:
1. Compute block hash
2. Create message header $\mathcal{H}_\mathcal{M}$
3. Sign $\mathcal{H}_\mathcal{M}$
4. Create $NewBlock$ message $\mathcal{M}$
5. Output $\mathcal{M}$


*Procedure*:
- $\eta_r = H_{SHA3-256}(\mathcal{B}_r^i.Header)$
- $\eta_{r-1} = H_{SHA3-256}(\mathcal{B}_{r-1}.Header)$
1. $\mathcal{H}_\mathcal{M} = (pk_N, r, s, \eta_r)$
    | Field       | Value    | 
    |-------------|----------|
    | $Signer$    | $pk_N$   |
    | $Round$     | $r$      |
    | $Step$      | $s$      |
    | $BlockHash$ | $\eta_r$ |
2. $`\sigma_\mathcal{M} = Sign_{BLS}(sk_N, \mathcal{H}_\mathcal{M})`$
3. $`\mathcal{M} = NewBlock(\mathcal{H}_\mathcal{M},\eta_{r-1},\mathcal{B}_r^i, \sigma_\mathcal{M})`$
    | Field       | Value                | 
    |-------------|----------------------|
    | $Header$    | $\mathcal{H}_\mathcal{M}$        |
    | $PrevHash$  | $\eta_{r-1}$         |
    | $Candidate$ | $\mathcal{B}_r^i$    |
    | $Signature$ | $\sigma_\mathcal{M}$ |
4. $output \text{ } \mathcal{M}$


<!-- FOOTNOTES -->
[^1]: In principle, a malicious block generator could create two valid candidate blocks. However, this case is automatically handled in the Reduction phase, since provisioners will reach agreement on a specific block hash.

<!--  -->
[mh]: ../README.md#consensus-message-header
[b]: ../../blockchain/README.md#block-structure
[ds]: ../sortition/README.md#deterministic-sortition-ds
# Blockchain
This section describes the formal definition of the Dusk chain, block, and transaction structures.

#### ToC
  - [**Chain**](#chain)
  - [`Block`](#block)
  - [`BlockHeader`](#blockheader)
  - [`Transaction`](#transaction)


## Candidate Block
A candidate block is the block generated in the [Proposal][prop] step by the provisioner extracted as block generator. This is the block on which other provisioners will have to reach an agreement. If an agreement is not reached by the end of the iteration, a new candidate block will be produced and a new iteration will start.

Therefore, for each iteration, only one (valid) candidate block can be produced[^1]. To reflect this, we denote a candidate block with $\mathsf{B}^c_{R,I}$, where $R$ is the consensus round, and $i$ is the consensus iteration. 
Note that we simplify this notation to simply $\mathsf{B}^c$ when this does not generate confusion.

A candidate block that reaches an agreement is called a *winning* block.



## **Chain**
The local copy of the blockchain stored by a node is defined as a vector of [blocks](#block) along with their *consensus status* label (see [Block Finality][fin]):

$$\textbf{Chain}: [(\mathsf{B}_0, Status_0), \text{ }\dots, (\mathsf{B}_n, Status_n)],$$

where $\mathsf{B}_0$ is the genesis block and $Status_0 = Final$ (that is, the genesis block is final by definition).

**Notation**
 - We use $\textbf{Chain}[i]$ to indicate the *i*th element of the chain.
 - We use $\mathsf{B} \in \textbf{Chain}$ to say a block $\mathsf{B}$ in our local chain
 - We use $\eta_\mathsf{B} \in \textbf{Chain}$ to say a block with hash $\eta_\mathsf{B}$ in our local chain
 - We use $\mathsf{B}_\eta$ to indicate the block with hash $\eta$.

<!-- TODO: define Genesis and Tip here; review use of Tip in procedures (we can use Chain instead) -->

## `Block`

| Field            | Type                   | Size      | Description        |
|------------------|------------------------|-----------|--------------------|
| $Header$         | [`BlockHeader`][bh]    | 112 bytes | Block header       |
| $Transactions$   | [`Transaction`][tx][ ] | variable  | Block transactions |

## `BlockHeader`

| Field                  | Type                    | Size       | Description                                         |
|------------------------|-------------------------|------------|-----------------------------------------------------|
| $Version$              | Unsigned Integer        | 8 bits     | Block version                                       |
| $Height$               | Unsigned Integer        | 64 bits    | Block height                                        |
| $Timestamp$            | Unsigned Integer        | 64 bits    | Block timestamp in Unix format                      |
| $GasLimit$             | Unsigned Integer        | 64 bits    | Block gas limit                                     |
| $Iteration$            | Unsigned Integer        | 8 bits     | Iteration at which the block was produced           |
| $PreviousBlock$        | Sha3 Hash               | 32 bytes   | Hash of the previous block                          |
| $Seed$                 | Signature               | 48 bytes   | Signature of the previous block's seed              |
| $Generator$            | Public Key              | 96 bytes   | Generator Public Key                                |
| $TransactionRoot$      | Blake3 Hash             | 32 bytes   | Root of transactions Merkle tree                    |
| $StateRoot$            | Sha3 Hash               | 32 bytes   | Root of contracts state Merkle tree                 |
| $PrevBlockCertificate$ | [`Attestation`][att]    | 112 bytes  | Certificate for the previous block                  |
| $FailedIterations$     | [`Attestation`][att][ ] | 0-28448 bytes (27.75 KB) | Aggregated votes of failed iterations |
| $Hash$                 | Sha3 Hash               | 32 bytes   | Hash of previous fields                             |
| $Attestation$          | [`Attestation`][att]    | 112 bytes  | Attestation of the $Valid$ votes for the block      |

The `BlockHeader` structure has a variable total size of 522 to 28970 bytes (28.8 KB).
This is reduced to 410-28448 bytes for a [*candidate block*][cb], since $Attestation$ is missing.

**Notation**

We denote the header of a block $\mathsf{B}$ as $\mathsf{H_B}$.

We define the *hash* of a block $\mathsf{B}$ as:

$\eta_\mathsf{B} = Hash_{SHA3}(Version||Height||Timestamp||GasLimit||Iteration||PreviousBlock||Seed||$
$\hspace{50pt}Generator||TransactionRoot||StateRoot||PrevBlockCertificate||FailedIterations)$

We also define the function $\eta(\mathsf{B})$ which computes $\eta_\mathsf{B}$ over a block $\mathsf{B}$.

## `Transaction`

| Field     | Type             | Size      | Description         |
|-----------|------------------|-----------|---------------------|
| $Version$ | Unsigned Integer | 32 bits   | Transaction version |
| $Type$    | Unsigned Integer | 32 bits   | Transaction type    |
| $Payload$ | Byte[ ]          | variable  | Transaction payload |


## Finality
Due to the asynchronous nature of the network, more than one block can reach consensus in the same round (but in different iterations), creating a chain *fork* (i.e., two parallel branches stemming from a common ancestor). This is typically due to consensus messages being delayed or lost due to network congestion.

When a fork occurs, network nodes can initially accept either of the two blocks at the same height, depending on which one they see first. 
However, when multiple same-height blocks are received, nodes always choose the lowest-iteration one. This mechanism allows to automatically resolve forks as soon as all conflicting blocks are received by all nodes.

As a consequence of the above, blocks from iterations greater than 0 could potentially be replaced if a lower-iteration block also reached consensus (see [*Fallback*][fal]). Instead, blocks reaching consensus at iteration 0 can't be replaced by lower-iteration ones with the same parent. However, they can be replaced if an ancestor block is reverted.

### Consensus State
To handle forks, we use the concept of Consensus State, which defines whether a block can or cannot be replaced by another one from the network.
In particular, Blocks in the [local chain][lc] can be in three states:

  - *Accepted*: the block has a $Valid$ quorum but there might be a lower-iteration block with the same parent that also reached a $Valid$ quorum; an Accepted block can then be replaced by a lower-iteration one; *Accepted* blocks are blocks that reached consensus at Iteration higher than 0 and for which not all previous iterations have a [Failed Attestation][atts]. 

  - *Attested*: the block has a Valid Quorum and all previous iterations have a Failed Attestation; this block cannot be replaced by a lower-iteration block with the same parent but one of its predecessors is Accepted and could be replaced; blocks reaching quorum at iteration 0 are Attested by definition (because no previous iteration exists).
  
  - *Final*: the block is Attested and all its predecessors are Final; this block is definitive and cannot be replaced in any case.

### Last Final Block
At any given moment, the local chain can be considered as made of two parts: a *final* one, from the genesis block to the last final block, and a *non-final* one, including all blocks after the last final block. Blocks in the non-final part can potentially be reverted until their state changes to Final (see [Rolling Finality][rf]. In contrast, the final part cannot be reverted in any way and is the definitive. When the chain tip is final, then the whole chain is final.

Due to its relevance, we formally define the ***last final block*** as the highest block in the local chain that has been marked as final, and denote it with $\mathsf{B}^f$.


### Rolling Finality
*Rolling Finality* is the mechanism by which non-final blocks become final.

The mechanism is based on the following observations:
 - Accepted blocks are the only potential "post-fork" blocks (i.e., the successor of a forking point), which can be replaced by a sibling (a block with the same parent). 
 - Considering an Accepted block $B^A$, a successor of $B^A$ being voted implicitly proves that a subset of the provisioner set, namely the ones that voted for it, have accepted $B^A$ into their chain. In other words, any attestation for a successor of $B^A$ implicitly confirms $B^A$ is in the local chain of a subset of provisioners.
 - Since each committee is randomly extracted with [Deterministic Sortition][ds], it can be considered as a random sampling of the provisioner set.
 - Each block added on top of the Accepted block $B^A$ increases the size of the random sampling of provisioners that accepted $B^A$, reducing the probability that other provisioners are working on a competing fork.
 - Each round/iteration executed after $B^A$ decreases the probability of a competing sibling being received. In other words, each iteration implies a certain time elapsed during which the competing block should have been received if it existed.
 - While all blocks succeeding $B^A$ include Attestations confirming $B^A$, only Attested blocks can be safely accounted for. In fact, Accepted blocks could also be replaced, making it hard to decide which number of Attestations are enough to consider $B^A$ as Final.

Considering only Attested blocks allow minimizing the risk of accounting for "confirmations" that are then replaced by a fork.

Based on the above, nodes follow the following rule: 
> Any accepted block $B^A$ in the local chain is marked as Final if 5 consecutive Attested blocks are accepted afterwards.

In other words, 5 consecutive Attested blocks finalize all previous Accepted blocks. In turn, the 5 Attested blocks also become Final (because the previous Accepted block is now Final), thus making the whole chain Final.

Note that this mechanism assumes that a block being finalized by the Rolling Finality has minimal probability of such a block being replaced.
<!-- TODO: Proper calculations are required to decide on the number of consecutive blocks and the actual probability -->

### Environment
<!-- TODO: RELAX_ITERATION_THRESHOLD = 10 -->

| Name               | Value          | Description                                          |
|--------------------|----------------|------------------------------------------------------|
| $RollingFinality$  | 5              | Number of Attested blocks for [Rolling Finality][rf] |

### Procedures

#### *GetBlockState*
The block state is computed according to the [Finality][fin] rules.

**Parameters**
- $\mathsf{B}$: the block being accepted to the chain

**Algorithm**
1. If all failed iterations have a [Failed Attestation][atts]:
   1. Set $cstate$ to "Attested"
   2. If $\mathsf{B}$'s parent is Final
      1. Set $cstate$ to "Final"
2. Otherwise, set $cstate$ to "Accepted"
3. Output $cstate$

**Procedure**

$\textit{GetBlockState}(\mathsf{B}):$
- $\texttt{set } h = \mathsf{H_B}.Height$
1. $\texttt{if } (|\mathsf{H_B}.FailedIterations| = \mathsf{H_B}.Iteration-1):$
   1. $\texttt{set } cstate = \text{"Attested"}$
   2. $\texttt{if } (\textbf{Chain}[h{-}1].State = \text{"Final"}) :$
      1. $\texttt{set } cstate = \text{"Final"}$
2. $\texttt{else } :$
   1. $\texttt{set } cstate = \text{"Accepted"}$
3. $\texttt{output } cstate$


#### *CheckRollingFinality*
This procedure checks if the last $RollingFinality$ blocks are all "Attested" and, if so, finalizes all non-final blocks.

**Procedure**
$\textit{CheckRollingFinality}():$
1. $rf =$ *HasRollingFinality*$()$
2. $\texttt{if } (rf = true) :$
   1. *MakeChainFinal*$()$

#### *HasRollingFinality*
This procedure outputs true if the last $RollingFinality$ are all Attested and false otherwise.

**Procedure**
$\textit{HasRollingFinality}():$
- $\texttt{set } tip = \mathsf{H}_{Tip}.Height$
1. $\texttt{for } i = tip \dots tip{-}RollingFinality :$
   1. $\texttt{if } \textbf{Chain}[i].State \ne \text{"Attested"}$
      1. $\texttt{output } false$
2. $\texttt{output } true$ 

#### *MakeChainFinal*
This procedure set to "Final" the state of all non-final blocks in $\textbf{Chain}$

<!----------------------- FOOTNOTES ----------------------->

[^1]: In principle, a malicious block generator could create two valid candidate blocks. However, this case is automatically handled in the Validation step, since provisioners will reach agreement on a specific block hash.


<!------------------------- LINKS ------------------------->
[b]:  #block
[bh]: #blockheader
[lc]: #chain
[tx]: #transaction

<!-- Basics -->
[cb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#candidate-block
[att]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#attestation
[sv]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#stepvotes

<!-- Chain Management -->
[fin]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#finality
[cs]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#consensus-state

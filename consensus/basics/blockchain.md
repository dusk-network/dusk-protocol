# Blockchain
This section formally describes the the Dusk blockchain and block finality.

**ToC**
  - [Blocks](#blocks)
      - [Candidate Block](#candidate-block)
      - [Full block](#full-block)
    - [Structures](#structures)
      - [`Block` Structure](#block-structure)
      - [`BlockHeader` Structure](#blockheader-structure)
  - [**Chain**](#chain)
      - [Main Chain](#main-chain)
      - [Local Chain](#local-chain)
        - [`ChainBlock` Structure](#chainblock-structure)
  - [Finality](#finality)
    - [Consensus State](#consensus-state)
    - [Last Final Block](#last-final-block)
    - [Rolling Finality](#rolling-finality)
    - [Environment](#environment)
    - [Procedures](#procedures)
      - [*GetBlockState*](#getblockstate)
      - [*CheckRollingFinality*](#checkrollingfinality)
      - [*HasRollingFinality*](#hasrollingfinality)
      - [*MakeChainFinal*](#makechainfinal)



## Blocks
#### Candidate Block
A candidate block is the block generated in the [Proposal][prop] step by the provisioner extracted as block generator. This is the block on which other provisioners will have to reach an agreement. If an agreement is not reached by the end of the iteration, a new candidate block will be produced and a new iteration will start.

Therefore, for each iteration, only one (valid) candidate block can be produced[^1]. To reflect this, we denote a candidate block with $\mathsf{B}^c_{R,I}$, where $R$ is the consensus round, and $i$ is the consensus iteration. 
Note that we simplify this notation to simply $\mathsf{B}^c$ when this does not generate confusion.

A candidate block that reaches an agreement is called a *winning* block.

#### Full block
A full block is a candidate block that reached a quorum agreement on both the [Validation][val] and [Ratification][rat]. It is essentially a candidate block with an [attestation][atts] of $Valid$ quorum.

### Structures
#### `Block` Structure

| Field          | Type                | Size                | Description        |
|----------------|---------------------|---------------------|--------------------|
| $Header$       | [`BlockHeader`][bh] | 522 Bytes - 28.8 KB | Block header       |
| $Transactions$ | `Transaction`[ ]    | variable            | Block transactions |

#### `BlockHeader` Structure

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

We also define the function $\eta(\mathsf{B})$ that given a block $\mathsf{B}$ outputs $\eta_\mathsf{B}$.

Finally, we use $\mathsf{B}_\eta$ to indicate the block with hash $\eta$.


## **Chain**
In each moment, it is possible that different nodes store a different version of the Dusk blockchain, which may vary in the extent of the chain (i.e., a difference in the height of last known block) or even composition (i.e., the last blocks are different, a case known as chain fork). For this reason, we distinguish between the chain on which the majority of the network agreed, referred to as the *main chain*, from the local version stored by a single node, referred to as the *local chain*.

#### Main Chain
The *main chain* is the most accepted sequence of blocks in the network. In particular, it is the chain containing the highest portion of the stakes. The main chain is not necessarily the longest one (a portion of provisioners might have independently produced more blocks) but it is the one that has the highest probability of moving forward.

The concept if main chain is typically used when assessing the validity of the chain known by a network node. Specifically, it is used to determine whether a node's chain has fallen behind the rest of the network or if it is on a separate branch (a chain fork). However, it is important to note that no single source of truth for the main chain can exist, and nodes can only compare their chain to those of other nodes. Based on this comparison, nodes choose what "seems" to be the main chain (see [Chain Management][cm]).

To help eliminate ambiguities and make node identify blocks that reached a definitive state, the concept of [*finality*][fin] is used. A block marked as *final* cannot be replaced and it is then permanently part of the main chain.

<!-- TODO: Describe the Genesis Block and its contents -->

#### Local Chain
The *local chain* is the copy of the blockchain stored by a node, and is defined as a vector of [blocks](#block) along with their *consensus state* label indicating their state with respect to [finality][fin] rules:

$$\textbf{Chain}: [(\mathsf{B}_{Genesis}, "Final"), \text{ }\dots, (\mathsf{B}_i, State_i)], \dots ,$$

where $\mathsf{B}_{Genesis}$ is the *Genesis Block*, $\mathsf{B}_i$ is the block at height $i$ and $State_i$ is its consensus status.

**Notation**
 - We use $\textbf{Chain}[i]$ to indicate the *i*th element of the chain.
 - We use $\mathsf{B} \in \textbf{Chain}$ to indicate that block $\mathsf{B}$ in the node's local chain.
 - We use $\eta_\mathsf{B} \in \textbf{Chain}$ to indicate that a block with hash $\eta_\mathsf{B}$ in the node's local chain.

Finally, we use $Tip$ to refer to the last block in the local chain.

##### `ChainBlock` Structure
For convenience, we define the `ChainBlock` structure as:
| Field            | Type         | Description                        |
|------------------|--------------|------------------------------------|
| $Block$          | [`Block`][b] | A full block accepted in the chain |
| $ConsensusState$ | String       | The block [consensus state][cs]    |

### System State
With the term *system state* we indicate the state of the underlying Virtual Machine (VM) resulting from the execution of all transactions included in the block of the local chain. In particular, we define a variable $VMState$ for each heigh $H$ (indicated as $VMState_H$).

#### `VMState` Structure
<!-- TODO: This should be expanded with more detailed info -->
This structure contains the information of the system state at a certain height $H$.

| Field          | Description                                     |
|----------------|-------------------------------------------------|
| $Provisioners$ | A full block accepted in the chain              |
| $Contracts$    | All existing contracts and their state (memory) |
| $Notes$        | Existing notes (both spent and unspent)         |

Note that this is an incomplete description of the system state, with several parts omitted, as they are irrelevant to the consensus protocol description.

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
   2. $\texttt{if } (\textbf{Chain}[h{-}1].ConsensusState = \text{"Final"}) :$
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
   1. $\texttt{if } \textbf{Chain}[i].ConsensusState \ne \text{"Attested"}$
      1. $\texttt{output } false$
2. $\texttt{output } true$ 

#### *MakeChainFinal*
This procedure set to "Final" the state of all non-final blocks in $\textbf{Chain}$

<!----------------------- FOOTNOTES ----------------------->

[^1]: In principle, a malicious block generator could create two valid candidate blocks. However, this case is automatically handled in the Validation step, since provisioners will reach agreement on a specific block hash.


<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md -->
[blk]: #blocks
[cb]:  #candidate-block
[fb]:  #full-block
[b]:   #block-structure
[bh]:  #blockheader-structure

[chn]: #chain
[mc]:  #main-chain
[lc]:  #local-chain
[chb]: #chainblock-structure
[sys]: #system-state
[vms]: #vmstate-structure

[fin]: #finality
[cs]:  #consensus-state
[lfb]: #last-final-block
[rf]:  #rolling-finality
[gbs]: #getblockstate
[crf]: #checkrollingfinality
[hrf]: #hasrollingfinality
[mcf]: #makechainfinal

<!-- Basics -->
[atts]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestations
[att]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestation
[sv]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#stepvotes

<!-- Protocol -->
[cenv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#environment

[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md

[ds]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md
[dsp]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md#deterministic-sortition-ds

[cm]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md
[fal]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#fallback




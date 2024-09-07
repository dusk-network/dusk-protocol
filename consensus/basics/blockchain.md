# Blockchain
This section formally describes the Dusk block, blockchain, and Rolling Finality.

**ToC**
  - [Blocks](#blocks)
      - [Candidate Block](#candidate-block)
      - [Full block](#full-block)
    - [Structures](#structures)
      - [`Block` Structure](#block-structure)
      - [`BlockHeader` Structure](#blockheader-structure)
    - [Procedures](#procedures)
      - [*VerifyBlock*](#verifyblock)
      - [*VerifyBlockHeader*](#verifyblockheader)
  - [**Chain**](#chain)
      - [Main Chain](#main-chain)
      - [Local Chain](#local-chain)
        - [`ChainBlock` Structure](#chainblock-structure)
        - [*AddToLocalChain*](#addtolocalchain)
    - [System State](#system-state)
      - [`VMState` Structure](#vmstate-structure)
  - [Rolling Finality](#rolling-finality)
    - [Consensus State](#consensus-state)
    - [Finality Rules](#finality-rules)
    - [Last Final Block](#last-final-block)
    - [Procedures](#procedures-1)
      - [*GetPNI*](#getpni)
      - [*GetConsensusState*](#getconsensusstate)
      - [*ApplyRollingFinality*](#applyrollingfinality)
      - [*UpdateConfirmedBlocks*](#updateconfirmedblocks)
      - [*UpdateFinalBlocks*](#updatefinalblocks)



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

| Field          | Type                | Size             | Description        |
|----------------|---------------------|------------------|--------------------|
| $Header$       | [`BlockHeader`][bh] | 522 - 1418 bytes | Block header       |
| $Transactions$ | `Transaction`[ ]    | variable         | Block transactions |
| $Faults$       | [`Fault`][fau][ ]   | variable         | Fault proofs       |

#### `BlockHeader` Structure

| Field                  | Type                    | Size        | Description                                    |
|------------------------|-------------------------|-------------|------------------------------------------------|
| $Version$              | Unsigned Int            | 8 bits      | Block version                                  |
| $Height$               | Unsigned Int            | 64 bits     | Block height                                   |
| $Timestamp$            | Unsigned Int            | 64 bits     | Block timestamp in Unix format                 |
| $GasLimit$             | Unsigned Int            | 64 bits     | Block gas limit                                |
| $Iteration$            | Unsigned Int            | 8 bits      | Iteration at which the block was produced      |
| $PreviousBlock$        | [SHA3][hash]            | 32 bytes    | Hash of the previous block                     |
| $Seed$                 | BLS Signature           | 48 bytes    | Signature of the previous block's seed         |
| $Generator$            | BLS Public Key          | 96 bytes    | Generator Public Key                           |
| $TransactionRoot$      | [Blake3][hash]          | 32 bytes    | Root of transactions Merkle tree               |
| $FaultRoot$            | [Blake3][hash]          | 32 bytes    | Root of faults Merkle tree                     |
| $StateRoot$            | [SHA3][hash]            | 32 bytes    | Root of contracts state Merkle tree            |
| $PrevBlockCertificate$ | [`Attestation`][att]    | 152 bytes   | [Certificate][cert] for the previous block     |
| $FailedIterations$     | [`Attestation`][att][ ] | 0-896 bytes | Fail Attestations of previous iterations       |
| $Hash$                 | [SHA3][hash]            | 32 bytes    | Hash of previous fields                        |
| $Attestation$          | [`Attestation`][att]    | 152 bytes   | Attestation of the $Valid$ votes for the block |

The `BlockHeader` structure has a variable total size of 634 to 1530 bytes.
This is reduced to 482-1378 bytes for a [*candidate block*][cb], since the $Attestation$ field is empty.

**Notation**

We denote the header of a block $\mathsf{B}$ as $\mathsf{H_B}$.

We define the *hash* of a block $\mathsf{B}$ as:

$\eta_\mathsf{B} = $[*SHA3*][hash]$(Version||Height||Timestamp||GasLimit||Iteration||PreviousBlock||Seed||$
$\hspace{50pt}Generator||TransactionRoot||FaultRoot||StateRoot||PrevBlockCertificate||FailedIterations)$

We also define the function $\eta(\mathsf{B})$ that given a block $\mathsf{B}$ outputs $\eta_\mathsf{B}$.

Finally, we use $\mathsf{B}_\eta$ to indicate the block with hash $\eta$.


### Procedures
We define the following Block-related procedures: 
  - [*VerifyBlock*][vb]: verifies the validity of a [`Block`][b] with respect to its parent block
  - [*VerifyBlockHeader*][vbh]: verifies the validity of a [`BlockHeader`][bh]

#### *VerifyBlock*
This procedure verifies a block is a valid successor of another block $\mathsf{B}^p$ (commonly, the $Tip$) and contains a [Success Attestation][atts]. If both conditions are met, it returns $true$, otherwise, it returns $false$.

If the block is an [Emergency Block][eb], the block $Attestation$ and $FailedIterations$ are not verified.

**Parameters**
- $\mathsf{B}$: the block to verify
- $\mathsf{B}^p$: the alleged $\mathsf{B}$'s parent

**Algorithm**
1. Verify $\mathsf{B}$'s header ([*VerifyBlockHeader*][vbh])
2. Verify $\mathsf{B}^p$'s certificate $\mathsf{A}_{\mathsf{B}^p}$ ([*VerifyAttestation*][va])
3. If $\mathsf{B}$ is a valid Emergency Block: output $true$
4. Verify $\mathsf{B}$'s attestation $\mathsf{A_B}$ ([*VerifyAttestation*][va])
5. For each attestation $\mathsf{A}_i$ in $FailedIterations$
   1. Verify $\mathsf{A}_i$ ([*VerifyAttestation*][va])
6.  If all verifications succeeded, output $true$

**Procedure**

$\textit{VerifyBlock}(\mathsf{B}):$
- $\textit{set }:$
  - $\mathsf{A}_{\mathsf{B}^p} = \mathsf{B}.PrevBlockCertificate$
  - $\mathsf{A_B} = \mathsf{B}.Attestation$
  - $\mathsf{CI} = {\mathsf{B}.PrevBlock,\mathsf{B}.Round,\mathsf{B}.Iteration}$
  - $\mathsf{CI}^p = {\mathsf{B}^p.PrevBlock,\mathsf{B}^p.Round,\mathsf{B}^p.Iteration}$
  
1. $\texttt{if } ($[*VerifyBlockHeader*][vbh]$(\mathsf{B}^p,\mathsf{B}) = false): \texttt{output } false$
2. $\texttt{if } ($[*VerifyAttestation*][va]$`(\mathsf{CI}^p, \mathsf{A}_{\mathsf{B}^p}, Success) = false): \texttt{output } false`$
3. $\texttt{if } ($[*isEmergencyBlock*][ieb]$(\mathsf{B}) = true): \texttt{output } true$
4. $\texttt{if } ($[*VerifyAttestation*][va]$`(\mathsf{CI}, \mathsf{A_B}, Success) = false): \texttt{output } false`$
5. $\texttt{for } i = 0 \dots |\mathsf{B}.FailedIterations|$
   - $\mathsf{A}_i = \mathsf{B}.FailedIterations[i]$
   1. $\texttt{if } (\mathsf{A}_i \ne NIL) :$
      - $\mathsf{CI}_i = (\mathsf{B}.PrevBlock,\mathsf{B}.Round,i)$
      1. $\texttt{if } ($[*VerifyAttestation*][va]$(\mathsf{CI}_i, \mathsf{A}_i, Fail) = false): \texttt{output } false$
6.  $\texttt{output } true$

#### *VerifyBlockHeader*
This procedure returns $true$ if all block header fields are valid with respect to the previous block and the included transactions. If so, it outputs $true$, otherwise, it outputs $false$.

Note that we require a minimum of $MinBlockTime$ (currently 10 seconds) between blocks. The candidate's timestamp is checked accordingly. Moreover, we reject blocks with correct timestamp that are published in advance. This implicitly assume nodes have a correct local time (ideally, synced through NTP).

**Parameters**
- $\mathsf{B}$: block to verify
- $\mathsf{B}^p$: previous block

**Algorithm**
1. If $\mathsf{B}$'s $Version$ is $Version$
2. And $\mathsf{B}$'s $Hash$ is the header's hash
3. And $\mathsf{B}$'s $Height$ is $\mathsf{B}^p$'s height plus 1
4. And $\mathsf{B}$'s $PrevBlock$ is $\mathsf{B}^p$'s hash
5. And $\mathsf{B}$'s $Seed$ is the generator's signature of $\mathsf{B}^p$'s $Seed$
6. And $\mathsf{B}$'s $Timestamp$ is at least 10 seconds after $\mathsf{B}^p$
7. And $\mathsf{B}$'s $Timestamp$ is lower than the local time
8. And $\mathsf{B}$'s transaction root is correct with respect to the transaction set
9. And $\mathsf{B}$'s fault root is correct with respect to the fault set
10. And $\mathsf{B}$'s state hash corresponds to the result of the state transition over $\mathsf{B}^p$
   1. Output $true$
11. Otherwise, output $false$

**Procedure**

$\textit{VerifyBlockHeader}(\mathsf{B}, \mathsf{B}^p)$:
- $SystemState_{\mathsf{B}^p} =$ [*ExecuteStateTransition*][est]$(SystemState_{\mathsf{B}^p}, \mathsf{B}.Transactions, BlockGas, pk_{G_\mathsf{B}})$
- $\texttt{if }$
  1. $(\mathsf{B}.Version = Version)$ 
  2. $\texttt{and } (\mathsf{B}.Hash = \eta(\mathsf{B}))$
  3. $\texttt{and } (\mathsf{B}.Height = \mathsf{B}^p.Height)+1$
  4. $\texttt{and } (\mathsf{B}.PreviousBlock = \mathsf{B}^p.Hash)$
  5. $\texttt{and } (\mathsf{B}.Seed = $*Sign*$(\mathsf{B}.Generator, \mathsf{B}^p.Seed))$
  6. $\texttt{and } (\mathsf{B}.Timestamp \gt \mathsf{B}^p.Timestamp+MinBlockTime)$
  7. $\texttt{and } (\mathsf{B}.Timestamp \le \tau_{Now}+TimestampMargin)$
  8. $\texttt{and } (\mathsf{B}.TransactionRoot = MerkleTree(\mathsf{B}.Transactions).Root)$
  9. $\texttt{and } (\mathsf{B}.FaultRoot = MerkleTree(\mathsf{B}.Faults).Root)$
  10. $\texttt{and } (\mathsf{B}.StateRoot = MerkleTree(SystemState_{\mathsf{B}^p}).Root):$
     1. $\texttt{output } true$

  11. $\texttt{else: output } false$


<p><br></p>

## **Chain**
In each moment, it is possible that different nodes store a different version of the Dusk blockchain, which may vary in the extent of the chain (i.e., a difference in the height of last known block) or even composition (i.e., the last blocks are different, a case known as chain fork). For this reason, we distinguish between the chain on which the majority of the network agreed, referred to as the *main chain*, from the local version stored by a single node, referred to as the *local chain*.

#### Main Chain
The *main chain* is the most accepted sequence of blocks in the network. In particular, it is the chain containing the highest portion of the stakes. The main chain is not necessarily the longest one (a portion of provisioners might have independently produced more blocks) but it is the one that has the highest probability of moving forward.

The concept if main chain is typically used when assessing the validity of the chain known by a network node. Specifically, it is used to determine whether a node's chain has fallen behind the rest of the network or if it is on a separate branch (a chain fork). However, it is important to note that no single source of truth for the main chain can exist, and nodes can only compare their chain to those of other nodes. Based on this comparison, nodes choose what "seems" to be the main chain (see [Chain Management][cm]).

To help eliminate ambiguities and make node identify blocks that reached a definitive state, the concept of [*finality*][rf] is used. A block marked as *final* cannot be replaced and it is then permanently part of the main chain.

<!-- TODO: Describe the Genesis Block and its contents -->

#### Local Chain
The *local chain* is the copy of the blockchain stored by a node, and is defined as a vector of [blocks](#block) along with their *consensus state* label indicating their state with respect to [Rolling Finality][rf] rules:

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


##### *AddToLocalChain*
This procedure takes a block $\mathsf{B}$, sets its $ConsensusState$ label, and adds it as the new tip of the local chain. Then it updates block states according to Rolling Finality.

**Parameters**
- $\mathsf{B}$: the block to add

**Procedure**
1. $ConsensusState =$ [*GetConsensusState*][gcs]$(\mathsf{B})$
2. $\textbf{Chain}[\mathsf{B}.Height]=(\mathsf{B}, ConsensusState)$
3. [*ApplyRollingFinality*][arf]$()$



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

<p><br></p>

-----

## Rolling Finality
Due to the asynchronous nature of the network, more than one block can reach consensus in the same round (but in different iterations), creating a chain *fork* (i.e., two parallel branches stemming from a common ancestor). This is typically due to consensus messages being delayed or lost due to network congestion.

When a fork occurs, network nodes can initially accept either of the two blocks at the same height, depending on which one they see first. 
However, when multiple same-height blocks are received, nodes always choose the lowest-iteration one. This mechanism allows to automatically resolve forks as soon as all conflicting blocks are received by all nodes.

Therefore, blocks from iterations greater than 0 could potentially be replaced if a lower-iteration block also reached consensus (see [*Fallback*][fal]). Instead, blocks reaching consensus at iteration 0 can't be replaced by lower-iteration ones with the same parent. However, they can be replaced if an ancestor block is reverted.

To help nodes (and users) determine when blocks are considered as definitively part of the blockchain, we employ the concept of block *finality*: a final block is a block that cannot be replaced in the local chain. In particular, we use the concept of *Rolling Finality*, where finality is determined by the conditions of the block itself (i.e. what Attestations it has) and the blocks before and after it. In other words, *Rolling Finality* is the mechanism through which blocks in the [local chain][lc] are marked as *final*.

Note that when a block is marked as final by the Rolling Finality, the probability of a higher priority block having reached a $Valid$ quorum is assumed to be negligible[^2].


**Rationale**

The mechanism is based on the following observations:
 - The only blocks that can be replaced by "higher-priority" sibling (a block with the same parent) are those with $Iteration \gt 0$ for which not all previous iterations have a [Fail Attestation][atts]. In particular, the number of such competing siblings corresponds to the number of previous iterations with no Attestation (that is, whose result is unknown).
 - Considering a replaceable block $B$, a successor of $B$ reaching a $Valid$ quorum implicitly proves that a set of the provisioners, namely those involved in its creation and voting, have accepted $B$ into their local chain. More specifically, any Attestation for a successor of $B$ implicitly confirms $B$ is in the local chain of a subset of the provisioner set. 
 - Since generators and committees are selected at random (see [Deterministic Sortition][ds]), they can be considered as a random sample of the provisioner set. Hence, an Attestation of a successor of $B$ implies that a large enough number of provisioners have $B$ in their chain.
 - Each new successor of $B$ (with an Attestation) increases the size of the random sample of provisioners that accepted $B$, reducing the probability that other provisioners are working on a competing fork.
 - Each round/iteration executed after $B$ decreases the probability of a competing sibling being received. In fact, each iteration implies a certain time elapsed during which the competing block, if it existed, should have been received.
 - While all attested successors of $B$ confirm $B$, only those at Iteration $0$, or having all previous iterations attested as Failed, can be safely accounted for its finality. In fact, other blocks can also be replaced, which make them unreliable to determine the finality status of $B$.


### Consensus State
To implement Rolling Finality, blocks in the [local chain][lc] are marked with a *Consensus State* label, which determines their current level of "replaceability".

In particular, blocks in the local chain can be in four states:

  - $Accepted$: the block has a Success Attestation but there might be a lower-iteration block that also reached a $Valid$ quorum; an $Accepted$ block can then be replaced if such a higher-priority block is received. Formally, a block is marked as $Accepted$ if it has  $Iteration \gt 0$ and not all previous iterations have a [Fail Attestation][atts]. 

  - $Attested$: the block has a Success Attestation and all previous iterations have a Fail Attestation; an $Attested$ block cannot be replaced by a lower-iteration block of the same round. Formally, a block is marked as $Attested$ if it has $Iteration = 0$ or all previous iterations have Fail Attestation.
  
  - $Confirmed$: the block is either $Accepted$ or $Attested$ and is confirmed by the [Finality rules][fr]; a $Confirmed$ block is unlikely to be replaced due to a number of block built on top of it; however, it might still be replaced if an ancestor is replaced.  

  - $Final$: the block is $Confirmed$ and its parent is $Final$; this block is definitive and cannot be replaced in any case. Note that a block can only be $Final$ if all ancestors are also $Final$.

### Finality Rules
The Consensus State of each block in the local chain is determined by the following rules:

 - A new block is marked as $Attested$ if $PNI=0$, where $PNI$ is the number of previous non-attested iterations, and $Accepted$ if $PNI>0$;
 - If a block is $Attested$, it is marked as $Confirmed$ if its successor is $Attested$ or $Confirmed$;
 - If a block is $Accepted$, it is marked as $Confirmed$ after $2 \times PNI$ consecutive $Attested$ or $Confirmed$ blocks;
 - If a block is $Confirmed$ and its parent is $Final$, it is marked as $Final$.

For instance, if a block has $Iteration = 5$ and only 2 previous iterations have a Fail Attestation, it is initially marked as $Accepted$ and becomes $Confirmed$ when the following $2 \times 2 = 4$ blocks are either $Attested$ or $Confirmed$. If, when marked as $Confirmed$, its parent block is $Final$, it is also marked as $Final$.

Note that the value of $PNI$ can be directly derived from the Attestations in the $FailedIterations$ field of a block.


### Last Final Block
At any given moment, the [local chain][lc] can be considered as made of two parts: a *final* one, from the genesis block to the last final block, and a *non-final* one, including all blocks after the last final block. Blocks in the non-final part can potentially be reverted until their state changes to $Final$. In contrast, the final part cannot be reverted in any way and is then definitive.

Due to its relevance, we formally define the ***last final block*** as the highest block in the local chain that has been marked as $Final$, and denote it with $\mathsf{B}^F$.

Note that, in the best case scenario, the *final* part of the chain includes all blocks except the tip. In fact, the tip can never be $Final$ as it has not been confirmed yet.


### Procedures

#### *GetPNI*
This procedure gets a block $\mathsf{B}$ and returns the number of previous non-attested iterations (PNI).

**Parameters**
- $\mathsf{B}$: the block being added to the chain

**Procedure**

$\textit{GetPNI}( \mathsf{B} ):$
- $\texttt{set } pni = 0$
1. $\texttt{for } i = 0 \dots \mathsf{B}.Iteration-1: $
   1. $\texttt{if } (\mathsf{H_B}.FailedIterations[i] = NIL):$
      1. $pni = pni + 1$
2. $\texttt{output } pni$

#### *GetConsensusState*
This procedure gets a block $\mathsf{B}$, checks previous non-attested iterations (PNI), and returns the label $Accepted$ or $Attested$ accordingly.

**Parameters**
- $\mathsf{B}$: the block being added to the chain

**Procedure**

$\textit{GetConsensusState}( \mathsf{B} ):$
1. $\texttt{set } pni =$ [*GetPNI*][pni]$(\mathsf{B})$
2. $\texttt{if } (pni > 0):$
   1. $\texttt{output } Accepted$
3. $\texttt{else } :$
   1. $\texttt{output } Attested$

#### *ApplyRollingFinality*
This procedure updates the labels of the non-final portion of the local chain, from the [Last Final Block][lfb] to the Tip, according to the [Finality rules][fr].

The update takes place only if the new Tip is labeled as $Attested$, since $Accepted$ blocks do not count for Rolling Finality.

**Procedure**

$\textit{ApplyRollingFinality}():$
1. $\texttt{if } ( \textbf{Chain}[tiph].ConsensusState = Attested ):$
   1. [*UpdateConfirmedBlocks*][ucb]$()$
   2. [*UpdateFinalBlocks*][ufb]$()$


#### *UpdateConfirmedBlocks*
This procedure iterates from the $Tip$ to the [Last Final Block][lfb] ($\mathsf{B}^F$) updates $Confirmed$ blocks.

The procedure uses a *Rolling Finality* ($rf$) counter to count the number of consecutive $Confirmed$ or $Attested$ blocks. This counter is initialized to 1 as it assumes the $Tip$ has been labeled as $Attested$. 
Note that $Attested$ blocks have $PNI=0$, so the check against the $rf$ counter always succeeds. In other words, if the loop reached an $Attested$ block, this is marked as $Confirmed$.

**Algorithm**
   1. Initialize $rf$ to 1
   2. For each block $\mathsf{B}$ between $Tip-1$ and $\mathsf{B}^F$:
      1. If $\mathsf{B}$ is $Confirmed$:
         1. Increase $rf$
      2. Otherwise:
         1. Get $\mathsf{B}$'s $PNI$ value
         2. If $rf$ is more than $PNI \times 2$:
            1. Mark $\mathsf{B}$ as $Confirmed$
         3. Otherwise, stop

**Procedure**

$\textit{UpdateConfirmedBlocks}():$
- $\texttt{set}:$
  - $tiph = Tip.Height$
  - $lfbh = \mathsf{B}^F.Height$
 1. $rf = 1$ 
 2. $\texttt{for } h = tiph-1 \text{ }\dots\text{ } lfbh+1 :$
    1. $\texttt{if } ( \textbf{Chain}[h].ConsensusState = Confirmed ):$
       1. $rf = rf + 1$
    2. $\texttt{else }:$
       1. $PNI =$ [*GetPNI*][pni]$(\textbf{Chain}[h].Block)$
       2. $\texttt{if } (rf \ge PNI \times 2):$
          1. $\textbf{Chain}[h].ConsensusState = Confirmed$
       3. $\texttt{else}: \texttt{break}$

#### *UpdateFinalBlocks*
This procedure iterates from the first non-final block to the Tip and marks as $Final$ all $Confirmed$ blocks following a $Final$ block.

**Procedure**

$\textit{UpdateFinalBlocks}():$
- $\texttt{set}:$
  - $tiph = Tip.Height$
  - $lfbh = \mathsf{B}^F.Height$
1. $\texttt{for } h = lfbh+1 \dots tiph :$
   1. $\texttt{if } ( \textbf{Chain}[h].ConsensusState = Confirmed ):$
      1. $\textbf{Chain}[h].ConsensusState = Final$
   2. $\texttt{else}: \texttt{break}$

<!----------------------- FOOTNOTES ----------------------->

[^1]: In principle, a malicious block generator could create two valid candidate blocks. However, this case is automatically handled in the Validation step, since provisioners will reach agreement on a specific block hash.

[^2]: While this assumption is secure enough, and confirmed empirically, proper calculations are due in the future to confirm the non-negligibility of such probability. A possible consequence of such assumption being non-true would be an increase in the number of blocks being required to be marked as *final*.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md -->
[blk]: #blocks
[cb]:  #candidate-block
[fb]:  #full-block
[b]:   #block-structure
[bh]:  #blockheader-structure
[vb]:  #verifyblock
[vbh]: #verifyblockheader


[chn]: #chain
[mc]:  #main-chain
[lc]:  #local-chain
[alc]: #AddToLocalChain
[chb]: #chainblock-structure
[sys]: #system-state
[vms]: #vmstate-structure

[rf]:  #rolling-finality
[cs]:  #consensus-state
[fr]:  #finality-rules
[lfb]: #last-final-block
[pni]: #GetPNI
[gcs]: #GetConsensusState
[arf]: #ApplyRollingFinality
[ucb]: #UpdateConfirmedBlocks
[ufb]: #UpdateFinalBlocks


<!-- Notation -->
[hash]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/notation.md#hash-functions

<!-- Basics -->
[cert]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#block-certificate
[atts]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestations
[att]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestation
[va]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#verifyattestation

[fau]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#faults

<!-- Protocol -->
[cenv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#environment
[eb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#emergencyblock
[ieb]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#isemergencyblock

[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md

[ds]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md

[cm]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md
[fal]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#fallback

<!-- TODO -->
[est]:    https://github.com/dusk-network/dusk-protocol/tree/main/virtual-machine

<!-- TODO: RELAX_ITERATION_THRESHOLD = 10 -->
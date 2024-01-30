# Chain Management
*Chain management* refers to how the SA protocol handles the local chain with respect to consensus-produced winning blocks, fork and out-of-sync events.

In particular, this section describes how new blocks are accepted to the local blockchain and how this chain can be updated when receiving valid blocks from the network.

### ToC
  - [Overview](#overview)
  - [Finality](#finality)
    - [Consensus State](#consensus-state)
    - [Rolling Finality](#rolling-finality)
  - [Environment](#environment)
  - [Block Verification](#block-verification)
    - [VerifyBlock](#verifyblock)
    - [VerifyBlockHeader](#verifyblockheader)
    - [*VerifyCertificate*](#verifycertificate)
    - [*VerifyVotes*](#verifyvotes)
  - [Block Management](#block-management)
    - [*HandleQuorum*](#handlequorum)
    - [*MakeWinning*](#makewinning)
    - [*AcceptBlock*](#acceptblock)
    - [GetBlockState](#getblockstate)
    - [CheckRollingFinality](#checkrollingfinality)
      - [HasRollingFinality](#hasrollingfinality)
      - [MakeChainFinal](#makechainfinal)
    - [HandleBlock](#handleblock)
  - [Fallback](#fallback)
    - [Fallback Procedure](#fallback-procedure)
  - [Synchronization](#synchronization)
    - [Procedures](#procedures)
      - [*SyncBlock*](#syncblock)
      - [*StartSync*](#startsync)
      - [*HandleSyncTimeout*](#handlesynctimeout)
      - [*AcceptPoolBlocks*](#acceptpoolblocks)


## Overview
At node level, there are two ways for blocks to be added to the [local chain][lc]: being the winning candidate of a [consensus round][sa] or being a certified block from the network. In the first case, the candidate block, for which a quorum has been reached in both [Validation][val] and [Ratification][rat] steps, becomes a *winning block* and is therefore added to the chain as the new tip (see [*AcceptBlock*][ab]).
In the latter case, a block is received from the network, which has already reached consensus (proved by the block [certificate][certs]). 

There are two main reasons a network block is not in the chain:
 
 - *out-of-sync*: the node fell behind the main chain, that is, its $Tip$ is part of the main chain but is at a lower height than the main chain's tip; in this case, the node needs to retrieve missing blocks to catch up with the network. This process is called [*synchronization*][syn] and is run with the peer that sent the triggering block.

 - *fork*: two or more candidates reached consensus in the same round (but different iteration); in this case, the node must first decide whether to stick to the local chain or switch to the other one. To switch, the node runs a [*fallback*][fal] process that reverts the local chain (and state) to the last finalized block and then starts the synchronization procedure to catch up with the main chain.

Incoming blocks (transmitted via [Block][bmsg] messages) are handled by the [*HandleBlock*][hb] procedure, which can trigger the [*Fallback*][fal] and [*SyncBlock*][sb] procedures to manage forks and out-of-sync cases, respectively.

## Finality
<!-- TODO: mv to Basics ? -->
Due to the asynchronous nature of the network, more than one block can reach consensus in the same round (but in different iterations), creating a chain *fork* (i.e., two parallel branches stemming from a common ancestor). This is typically due to consensus messages being delayed or lost due to network congestion.

When a fork occurs, network nodes can initially accept either of the two blocks at the same height, depending on which one they see first. 
However, when multiple same-height blocks are received, nodes always choose the lowest-iteration one. This mechanism allows to automatically resolve forks as soon as all conflicting blocks are received by all nodes.

As a consequence of the above, blocks from iterations greater than 0 could potentially be replaced if a lower-iteration block also reached consensus (see [*Fallback*][fal]). Instead, blocks reaching consensus at iteration 0 can't be replaced by lower-iteration ones with the same parent. However, they can be replaced if an ancestor block is reverted.

### Consensus State
To handle forks, we use the concept of Consensus State, which defines whether a block can or cannot be replaced by another one from the network.
In particular, Blocks in the [local chain][lc] can be in three states:

  - *Accepted*: the block has a $Valid$ quorum but there might be a lower-iteration block with the same parent that also reached a $Valid$ quorum; an Accepted block can then be replaced by a lower-iteration one; *Accepted* blocks are blocks that reached consensus at Iteration higher than 0 and for which not all previous iterations have a [Failed Certificate][certs]. 

  - *Attested*: the block has a Valid Quorum and all previous iterations have a Failed Certificate; this block cannot be replaced by a lower-iteration block with the same parent but one of its predecessors is Accepted and could be replaced; blocks reaching quorum at iteration 0 are Attested by definition (because no previous iteration exists).
  
  - *Final*: the block is Attested and all its predecessors are Final; this block is definitive and cannot be replaced in any case.

**Final and Non-Final**
At any given moment, the local chain can be considered as made of two parts: a *final* one, from the genesis block to the last final block, and a *non-final* one, including all blocks after the last final block. Blocks in the non-final part can potentially be reverted until their state changes to Final (see [Rolling Finality][rf]. In contrast, the final part cannot be reverted in any way and is the definitive. When the chain tip is final, then the whole chain is final.

Due to its relevance, we formally define the ***last final block*** as the highest block in the local chain that has been marked as final, and denote it with $\mathsf{B}^f$.


### Rolling Finality
*Rolling Finality* is the mechanism by which non-final blocks become final.

The mechanism is based on the following observations:
 - Accepted blocks are the only potential "post-fork" blocks (i.e., the successor of a forking point), which can be replaced by a sibling (a block with the same parent). 
 - Considering an Accepted block $B^A$, a successor of $B^A$ being voted implicitly proves that a subset of the provisioner set, namely the ones that voted for it, have accepted $B^A$ into their chain. In other words, any certificate for a successor of $B^A$ implicitly confirms $B^A$ is in the local chain of a subset of provisioners.
 - Since each committee is randomly extracted with [Deterministic Sortition][ds], it can be considered as a random sampling of the provisioner set.
 - Each block added on top of the Accepted block $B^A$ increases the size of the random sampling of provisioners that accepted $B^A$, reducing the probability that other provisioners are working on a competing fork.
 - Each round/iteration executed after $B^A$ decreases the probability of a competing sibling being received. In other words, each iteration implies a certain time elapsed during which the competing block should have been received if it existed.
 - While all blocks succeeding $B^A$ include Certificates confirming $B^A$, only Attested blocks can be safely accounted for. In fact, Accepted blocks could also be replaced, making it hard to decide which number of Certificates are enough to consider $B^A$ as Final.

Considering only Attested blocks allow minimizing the risk of accounting for "confirmations" that are then replaced by a fork.

Based on the above, nodes follow the following rule: 
> Any accepted block $B^A$ in the local chain is marked as Final if 5 consecutive Attested blocks are accepted afterwards.

In other words, 5 consecutive Attested blocks finalize all previous Accepted blocks. In turn, the 5 Attested blocks also become Final (because the previous Accepted block is now Final), thus making the whole chain Final.

Note that this mechanism assumes that a block being finalized by the Rolling Finality has minimal probability of such a block being replaced.
<!-- TODO: Proper calculations are required to decide on the number of consecutive blocks and the actual probability -->

## Environment
The environment for the block-processing procedures includes node-level parameters, conditioning the node's behavior during synchronization, and state variables that help keep track of known blocks and handle the synchronization protocol execution.

**Parameters**

| Name            | Value | Description                                     |
|-----------------|-------|-------------------------------------------------|
| $SyncTimeout$   | $5$   | Synchronization procedure timeout (in seconds)  |
| $MaxSyncBlocks$ | $500$ | Maximum number of blocks in a sync session      |

**State**
<!-- TODO: rename inSync to reflect its purpose. It should be "Syncing". After changing the name, switch true with false -->

| Name          | Type                 | Description                             |
|---------------|----------------------|-----------------------------------------|
| $BlockPool$   | $\mathsf{Block}$ [ ] | List of received blocks with height more than $Tip.Height +1$. |
| $Blacklist$   | $\mathsf{Block}$ [ ] | list of blacklisted blocks; a block is blacklisted if it has been replaced by a better block (e.g. a block with same height but lower iteration number). |
| $inSync$      | Boolean              | $false$ if we are running a synchronization procedure with some peer, $true$ otherwise. It is initially set to $true$ |
| $syncPeer$    | Peer ID              | If $inSync$ is false, it contains the ID of the peer the node is synchronizing with.    |
| $\tau_{Sync}$ | Timestamp            | if $inSync$ is false, it contains the time from when the $syncTimeout$ is checked against; in other words, it indicates either the starting time of the synchronization process, or the time we received and accepted the last block from $syncPeer$. |
| $syncFrom$    | Integer | The height of the starting block during the synchronization process. |
| $syncTo$      | Integer | The heights of last blocks of synchronization process.               |


## Block Verification
We here define the procedures to verify the validity of a block: [*VerifyBlock*][vb], [*VerifyBlockHeader*][vbh], [*VerifyCertificate*][vc], and [*VerifyVotes*][vv].

### VerifyBlock
The *VerifyBlock* procedure verifies a block is a valid successor of another block $\mathsf{B}^p$ (commonly, the $Tip$) and contains a valid certificate. If both conditions are met, it returns $true$, otherwise, it returns $false$.

***Parameters***
- $\mathsf{B}$: the block to verify
- $\mathsf{B}^p$: the alleged $\mathsf{B}$'s predecessor

***Algorithm***
1. Verify $\mathsf{B}$'s header ([*VerifyBlockHeader*][vbh])
2. If header's not valid, output $false$
3. Verify $\mathsf{B}^p$'s certificate $\mathsf{C}_\mathsf{B^p}$ ([*VerifyCertificate*][vc])
4. If $\mathsf{C}^p$ is not valid, output $false$
5. Verify $\mathsf{B}$'s certificate $\mathsf{C}_\mathsf{B}$ ([*VerifyCertificate*][vc])
6. If certificate's not valid, output $false$
7. For each certificate $\mathsf{C}_i$ in $FailedIterations$
   1. Verify $\mathsf{C}_i$ ([*VerifyCertificate*][vc])
8. If all verifications succeeded, output $true$

***Procedure***
$\textit{VerifyBlock}(\mathsf{B}):$
- $\textit{set }:$
  - $\mathsf{C}_{\mathsf{B}^p} = \mathsf{B}.PrevBlockCertificate$
  - $\mathsf{C}_\mathsf{B} = \mathsf{B}.Certificate$
  - $\upsilon_\mathsf{B} = (\mathsf{B}.PrevBlock,\mathsf{B}.Round,\mathsf{B}.Iteration,Valid,\eta_\mathsf{B})$
  - $\upsilon_{\mathsf{B}^p} = (\mathsf{B}^p.PrevBlock,\mathsf{B}^p.Round,\mathsf{B}^p.Iteration,Valid,\eta_{\mathsf{B}^p})$
1. $isValid$ = [*VerifyBlockHeader*][vbh]$(\mathsf{B}^p,\mathsf{B})$
2. $\texttt{if } (isValid = false): \texttt{output } false$
3. $isValid$ = [*VerifyCertificate*][vc]$`(\mathsf{C}_{\mathsf{B}^p},\upsilon_\mathsf{B})`$
4. $\texttt{if } (isValid = false): \texttt{output } false$
5. $isValid$ = [*VerifyCertificate*][vc]$`(\mathsf{C}_{\mathsf{B}},\upsilon_{\mathsf{B}^p})`$
6. $\texttt{if } (isValid = false): \texttt{output } false$
7. $\texttt{for } i = 0 \dots |\mathsf{B}.FailedIterations|$
   - $\mathsf{C}_i = \mathsf{B}.FailedIterations[i]$
   1. $\texttt{if } (\mathsf{C}_i \ne NIL) :$
      <!-- TODO: support Invalid/NoQuorum votes -->
      - $\upsilon_i = (\mathsf{B}.PrevBlock,\mathsf{B}.Round,i,NoCandidate)$
      - $isValid =$ [*VerifyCertificate*][vc]$(\mathsf{C}_i, \upsilon_i)$
      - $\texttt{if } (isValid = false): \texttt{output } false$
8.  $\texttt{output } true$

### VerifyBlockHeader
<!-- TODO: rename to VerifyCandidate ? -->
*VerifyBlockHeader* returns $true$ if all block header fields are valid with respect to the previous block and the included transactions. If so, it outputs $true$, otherwise, it outputs $false$.

***Parameters***
- $\mathsf{B}$: block to verify
- $\mathsf{B}^p$: previous block

***Algorithm***
1. Check $Version$ is $0$
2. Check $Hash$ is the header's hash
3. Check $Height$ is $\mathsf{B}^p$'s height plus 1
4. Check $PrevBlock$ is $\mathsf{B}^p$'s hash
5. Check $Timestamp$ is higher than $\mathsf{B}^p$'s timestamp
6. Check transaction root is correct with respect to the transaction set
7. Check state hash corresponds to the result of the state transition over $\mathsf{B}^p$
- If any check failed
  1. Output $false$
- Otherwise, output $true$

***Procedure***

$\textit{VerifyBlockHeader}(\mathsf{B}, \mathsf{B}^p)$:
- $newState =$ *ExecuteTransactions*$(State_{\mathsf{B}^p}, \mathsf{B}.Transactions), BlockGas, pk_{G_\mathsf{B}})$
- $\texttt{if }$
  1. $(\mathsf{B}.Version > 0)$ 
  2. $\texttt{or } (\mathsf{B}.Hash \ne$ *Hash*$`_{SHA3}(\mathsf{H}_{\mathsf{B}}))`$
  3. $\texttt{or } (\mathsf{B}.Height \ne \mathsf{B}^p.Height)$
  4. $\texttt{or } (\mathsf{B}.PreviousBlock \ne \mathsf{B}^p.Hash)$
  5. $\texttt{or } (\mathsf{B}.Timestamp \lt \mathsf{B}^p.Timestamp)$
  6. $\texttt{or } (\mathsf{B}.TransactionRoot \ne MerkleTree(\mathsf{B}.Transactions).Root)$
  7. $\texttt{or } (\mathsf{B}^c.StateRoot \ne newState.Root):$
     1. $\texttt{output } false$

   8. $\texttt{output } true$


### *VerifyCertificate*
*VerifyCertificate* checks a block's certificate by verifying the Validation and Ratification aggregated signatures against the respective committees.

***Parameters***
- $\mathsf{C}$: the certificate to verify
- $\upsilon$: the vote's data, containing: the previous block's hash, the iteration number, the winning vote, the block's hash

***Algorithm***
1. Check both Validation and Ratification votes are present
2. Verify Validation votes
3. If votes are not valid, output $false$
4. Verify Ratification votes
5. If votes are not valid, output $false$
6. Output $true$

***Procedure***

$\textit{VerifyCertificate}(\mathsf{C}, \upsilon):$
- $\texttt{set}:$
   - $`\mathsf{SV}^V, \mathsf{SV}^R \leftarrow \mathsf{C}`$
   - $\eta_{\mathsf{B}}^p, R, I, v, \eta_{\mathsf{B}} \leftarrow \upsilon$
   - $\mathcal{C}^V =$ [*ExtractCommittee*][ec]$(R,I, ValStep)$
   - $\upsilon^V = (\eta_{\mathsf{B}}^p||R||I||v||\eta_{\mathsf{B}}||ValStep)$ 
   - $\mathcal{C}^R =$ [*ExtractCommittee*][ec]$(R,I, RatStep)$
   - $\upsilon^R = (\eta_{\mathsf{B}}^p||R||I||v||\eta_{\mathsf{B}}||RatStep)$
   - $Q =$ [*GetQuorum*][gq]$(v)$
1. $\texttt{if } (\mathsf{SV}^V = NIL) \texttt{ or } (\mathsf{SV}^R = NIL):$
   1. $\texttt{output } false$
2. $isValid =$ [*VerifyVotes*][vv]$`(\mathsf{SV}^V, \upsilon^V, Q, \mathcal{C}^V)`$
3. $\texttt{if } (isValid{=}false): \texttt{output } false$
4. $isValid =$ [*VerifyVotes*][vv]$`(\mathsf{SV}^R, \upsilon^R, Q, \mathcal{C}^R)`$
5. $\texttt{if } (isValid{=}false): \texttt{output } false$
6. $\texttt{output } true$

### *VerifyVotes*
*VerifyVotes* checks the aggregated votes are valid and reach the target quorum.

***Parameters***
- $\mathsf{SV}$: $\mathsf{StepVotes}$ with the aggregated votes
- $\upsilon$: the [signature value][ms]
- $Q$: the target quorum
- $\mathcal{C}$: the step committee

***Algorithm***
1. Compute subcommittee $C^{\boldsymbol{bs}}$ from $\mathsf{SV}.BitSet$
2. If credits in $C^{\boldsymbol{bs}}$ are less than the target quorum $Q$
   1. Output $false$
3. Aggregate public keys of $C^{\boldsymbol{bs}}$ members
4. Verify aggregated signature over $\upsilon$

***Procedure***

$VerifyVotes(\mathsf{SV}, \upsilon, Q)$:
- $\texttt{set}:$
  - $\boldsymbol{bs}, \sigma_{\boldsymbol{bs}} \leftarrow \mathsf{SV}$
1. $\mathcal{C}^\boldsymbol{bs} =$ [*SubCommittee*][sc]$(\mathcal{C}, \boldsymbol{bs})$
2. $\texttt{if } ($[*CountCredits*][cc]$(\mathcal{C}, \boldsymbol{bs}) \lt Q):$
   1. $\texttt{output } false$
3. $pk_{\boldsymbol{bs}} = AggregatePKs(C^{\boldsymbol{bs}})$
4. $\texttt{output } Verify_{BLS}(\upsilon, pk_{\boldsymbol{bs}}, \sigma_{\boldsymbol{bs}})$


## Block Management
We here define block-management procedures: 
  - *HandleQuorum*: process a $\mathsf{Quorum}$ message from the network
  - *MakeWinning*: sets a candidate block as the winning block of the round
  - *AcceptBlock*: accept a block as the new tip
  - *HandleBlock*: process a block from the network

### *HandleQuorum*
<!-- TODO: check if this is explained somewhere
# Quorum Handler
When the [Validation][val] and [Ratification][rat] steps reach a quorum, a [Quorum][qmsg] message is produced, which contains a [Certificate][cert] with the aggregated votes of the two quorum committees.
Since the certificate proves a candidate reached a quorum, receiving this message is sufficient to accept the candidate into the local chain.

The Quorum handler task manages [Quorum][qmsg] messages produced by the node or received from the network. In particular, when such a message is received (or produced), the corresponding candidate block is accepted.

## Overview
When a node generates or receives a $\mathsf{Quorum}$ message, it adds the included $\mathsf{Certificate}$ to the corresponding candidate block and makes it a *winning block* for the corresponding round and iteration.

If the $\mathsf{Quorum}$ message was received from the network, the aggregated votes (see [`StepVotes`][sv]) are verified against the generator and committee members of the corresponding round and iteration. Both the validity of the votes and the quorum quota are verified.

The winning block will then be accepted as the new tip of the local chain.
 -->

*HandleQuorum* manages $\mathsf{Quorum}$ messages for the current round $R$. If the message was received from the network, it is first verified ([*VerifyQuorum*][va]) and then propagated.
The corresponding candidate block is then marked as the winning block of the round ([*MakeWinning*][mw]).

***Parameters***
- [SA Environment][cenv]
- $R$: round number

***Algorithm***
1. Loop:
   1. If a $\mathsf{Quorum}$ message $\mathsf{M}^Q$ is received for round $R$:
      1. Verify $\mathsf{M}^Q.Certificate$ ($\mathsf{C}$) is valid ([*VerifyQuorum*][va])
      2. If valid:
         1. Propagate $\mathsf{M}^Q$
         2. If the quorum vote is $Valid$
            1. Fetch candidate $\mathsf{B}^c$ from $\mathsf{M}^Q.BlockHash$
            2. If $\mathsf{B}^c$ is unknown, request it to peers ([*GetCandidate*][gcmsg])
            3. Set the winning block $\mathsf{B}^w$ to $\mathsf{B}^c$ with $\mathsf{M}^Q.Certificate$
         3. Otherwise
            1. Add $\mathsf{C}$ to the $\boldsymbol{FailedCertificates}$ list
         4. Stop [*SAIteration*][sai]

***Procedure***
<!-- TODO: define candidate pool/db and functions, eg FetchCandidate -->
$\textit{HandleQuorum}( R ):$
1. $\texttt{loop}$:   
   1.  $\texttt{if } (\mathsf{M}^Q =$ [*Receive*][mx]$(\mathsf{Quorum}, R) \ne NIL):$
       -  $\texttt{set}:$
          - $`\mathsf{CI}, \mathsf{VI}, \mathsf{C} \leftarrow \mathsf{M}^Q`$
          - $`\eta_{\mathsf{B}^p}, R_{\mathsf{M}}, I_{\mathsf{M}}, \leftarrow \mathsf{CI}`$
          - $`v, \eta_{\mathsf{B}^c} \leftarrow \mathsf{VI}`$
          - $\upsilon = (\eta_{\mathsf{B}^p}||I_{\mathsf{M}}||v||\eta_\mathsf{B})$

       1. $isValid =$ [*VerifyCertificate*][vc]$(\mathsf{C}, \upsilon)$
       2. $\texttt{if } (isValid = true) :$
          1. $\texttt{stop}$([*SAIteration*][sai])
          2. [*Propagate*][mx]$(\mathsf{M}^Q)$
          3. $\texttt{if } (v = Valid) :$
             1. $\mathsf{B}^c =$ *FetchCandidate* $(\eta_{\mathsf{B}^c})$
             2. $\texttt{if } (\mathsf{B}^c = NIL) :$
               - $\mathsf{B}^c =$ [*GetCandidate*][gcmsg]$(\eta_{\mathsf{B}^c})$
             3. [*MakeWinning*][mw]$(\mathsf{B}^c, \mathsf{C})$
          4. $\texttt{else } :$
             1. $\boldsymbol{FailedCertificates}[I_{\mathsf{M}}] = {\mathsf{C}}$


### *MakeWinning*
*MakeWinning* adds a certificate to a candidate block and sets the winning block variable $\mathsf{B}^w$.

***Parameters***
- $\mathsf{B}$: candidate block
- $\mathsf{C}$: block certificate

***Algorithm***
1. Add certificate $\mathsf{C}$ to candidate block $\mathsf{B}$
2. Set winning block $\mathsf{B}^w$ to $\mathsf{B}$

***Procedure***

$MakeWinning(\mathsf{B}, \mathsf{C}):$
1. $\mathsf{B}.Certificate = \mathsf{C}$
2. $\mathsf{B}^w = \mathsf{B}$


### *AcceptBlock*
*AcceptBlock* sets a block $\mathsf{B}$ as the new chain $Tip$. It also updates the local state accordingly by executing all transactions in the block and setting the $Provisioners$ state variable. 

***Parameters***
- $\mathsf{B}$: the block to accept as the new chain tip

***Algorithm***
1. Extract $Transactions$, $GasLimit$, and $Generator$ from block $\mathsf{B}$
2. Generate new state ($newState$) by applying $Transactions$ on the current $State$, and assigning the block reward to $Generator$
3. Update the $Provisioners$ set
4. Set $Tip$ to block $\mathsf{B}$
5. Compute the consensus state $s$ of $\mathsf{B}$
6. Add $(Tip, s)$ to the local chain
7. Check Rolling Finality

***Procedure***

$\textit{AcceptBlock}(\mathsf{B}):$
1. $\texttt{set }$:
   - $\boldsymbol{txs} = \mathsf{B}.Transactions$
   - $gas = \mathsf{B}.GasLimit$
   - $pk_{\mathcal{G}} = \mathsf{B}.Generator$
   - $h = \mathsf{H_B}.Height$
2. $newState =$ *ExecuteTransactions*$(State, \boldsymbol{txs}, gas, pk_{\mathcal{G}})$
3. $Provisioners = newState.Provisioners$
4. $Tip = \mathsf{B}$
5. $s =$ *GetBlockState*$(\mathsf{B})$
6. $\textbf{Chain}[h]=(\mathsf{B}, s)$
7. [*CheckRollingFinality*][crf]$()$

### GetBlockState
The block state is computed according to the [Finality][fin] rules.

***Parameters***
- $\mathsf{B}$: the block being accepted to the chain

***Algorithm***
1. If all failed iterations have a [Failed certificate][certs]:
   1. Set $cstate$ to "Attested"
   2. If $\mathsf{B}$'s parent is Final
      1. Set $cstate$ to "Final"
2. Otherwise, set $cstate$ to "Accepted"
3. Output $cstate$

***Procedure***

$\textit{GetBlockState}(\mathsf{B}):$
- $\texttt{set } h = \mathsf{H_B}.Height$
1. $\texttt{if } (|\mathsf{H_B}.FailedIterations| = \mathsf{H_B}.Iteration-1):$
   1. $\texttt{set } cstate = \text{"Attested"}$
   2. $\texttt{if } (\textbf{Chain}[h{-}1].State = \text{"Final"}) :$
      1. $\texttt{set } cstate = \text{"Final"}$
2. $\texttt{else } :$
   1. $\texttt{set } cstate = \text{"Accepted"}$
3. $\texttt{output } cstate$


### CheckRollingFinality
*CheckRollingFinality* checks if the last $RollingFinality$ blocks are all "Attested" and, if so, finalizes all non-final blocks.

***Procedure***
$\textit{CheckRollingFinality}():$
1. $rf =$ *HasRollingFinality*$()$
2. $\texttt{if } (rf = true) :$
   1. *MakeChainFinal*$()$

#### HasRollingFinality
*HasRollingFinality* outputs true if the last $RollingFinality$ are all Attested and false otherwise.

***Procedure***
$\textit{HasRollingFinality}():$
- $\texttt{set } tip = \mathsf{H}_{Tip}.Height$
1. $\texttt{for } i = tip \dots tip{-}RollingFinality :$
   1. $\texttt{if } \textbf{Chain}[i].State \ne \text{"Attested"}$
      1. $\texttt{output } false$
2. $\texttt{output } true$ 

#### MakeChainFinal
*MakeChainFinal* set to "Final" the state of all non-final blocks in $\textbf{Chain}$

### HandleBlock
The *HandleBlock* procedure processes a full block received from the network and decides whether to trigger the synchronization or fallback procedures.
The procedure acts depending on the block's height: if the block has the same height as the $Tip$, but lower $Iteration$, it starts the [*Fallback*][fal] procedure; if the block's height is more than $Tip.Height+1$, it executes the [*SyncBlock*][sb] is executed to start or continue the synchronization process; if the block as height lower than the $Tip$, the block is discarded.

***Parameters*** 
- $\mathsf{M}^{Block}$: the incoming $\mathsf{Block}$ message

***Algorithm***
1. Loop:
   1. If a $\mathsf{Quorum}$ message $\mathsf{M}^Q$ is received for round $R$:
      - Extract the block $\mathsf{B}$ and the message sender $\mathcal{S}$
      1. Check block hash $\mathsf{H_B}$
      2. Check if block is blacklisted
      3. If block height is same as $Tip$
         1. Check block's validity as successor of the $Tip$'s predecessor
         2. If the block is valid
         3. And block's iteration is lower than $Tip$
         4. And block is not $Tip$
            1. Start *fallback* procedure ([*Fallback*][fal])
      4. If block height is lower than $Tip$
         1. Discard block 
      5. If block height is higher than $Tip$
         1. Start *synchronization* ([*SyncBlock*][sb])

***Procedure***

$\textit{HandleBlock}():$
1. $\texttt{loop}$:   
   1.  $\texttt{if } (\mathsf{M}^{Block} =$ [*Receive*][mx]$(\mathsf{Block}) \ne NIL):$
       - $\mathsf{B},\mathcal{S} \leftarrow \mathsf{M}^{Block}$
       1. $isValidHash = (\mathsf{B}.Hash =$ *Hash*$`_{SHA3}(\mathsf{H_B}))`$ \
          $\texttt{if } (isValidHash = false) : \texttt{stop}$
       2. $\texttt{if } (\mathsf{B} \in Blacklist) : \texttt{stop}$
          <!-- B.Height = Tip.Height -->
       3. $\texttt{if } (\mathsf{B}.Height = Tip.Height) :$
          1. $isValid = $[*VerifyBlock*][vb]$(\mathsf{B}, \mathsf{B}_{Tip.Height-1})$
          2. $\texttt{if } (isValid = true)$
          3. $\texttt{and } (\mathsf{B}.Iteration \ge Tip.Iteration)$
          4. $\texttt{and } (\mathsf{B} \ne Tip) :$
             1. [*Fallback*][fal]$()$
          <!-- B.Height < Tip.Height -->
       4. $\texttt{if } (\mathsf{B}.Height < Tip.Height) :$
          1. $\texttt{stop}$
          <!-- B.Height > Tip.Height -->
       5. $\texttt{if } (\mathsf{B}.Height > Tip.Height) :$
          1. [*SyncBlock*][sb]$(\mathsf{B}, \mathcal{S})$


## Fallback
The *Fallback* procedure reverts the local state to the last [finalized][fin] block. The procedure is triggered by [*HandleBlock*][hb] when receiving a block at the same height as the $Tip$ but with lower $Iteration$. 

In fact, this event indicates that a quorum was reached on a previous candidate and that the node is currently on a fork. To guarantee nodes converge on a common block, we give priority to blocks produced at lower iterations.

### Fallback Procedure
The *Fallback* procedure first retrieves the last finalized block  $\mathsf{B}^f$ in the local chain; than it deletes all blocks from $Tip$ to $\mathsf{B}^f$ (excluded), pushing all their transactions back to the mempool. The local $State$ and $Provisioners$ are updated accordingly.

After reverting, the synchronization procedure is triggered to catch up with the main chain.

Note that the previous $Tip$ is added to the *Blacklist* because we are choosing a better fork.

***Algorithm***
1. Retrieve last final block $\mathsf{B}^f$
2. For each block between $Tip$ and $\mathsf{B}^f$
  1. Push $\mathsf{B}^f$'s transactions back to mempool
  2. Delete block $\mathsf{B}^f$
3. Blacklist $Tip$
4. Set $Tip$ to $\mathsf{B}^f$
5. Revert VM state to $\mathsf{B}^f$'s state
6. Update $Provisioners$
7. Start *synchronization* ([*SyncBlock*][sb])

***Procedure***

$\textit{Fallback}():$
1. $\mathsf{B}^f =$ *GetLastFinal*$()$
2. $\texttt{for } i = Tip.Height \dots \mathsf{B}^f.Height :$
  1. $Mempool = Mempool \cup \{ \mathsf{B}_i.Transactions \}$
  2. $Delete(\mathsf{B}_i)$
3. $Blacklist = Blacklist \cup Tip$
4. $Tip = \mathsf{B}^f$
5. $VM.Revert(\mathsf{B}^f)$
6. $Provisioners = VM.State.Provisioners$
7. [*SyncBlock*][sb]$(\mathsf{B}, \mathcal{S})$


## Synchronization
The *synchronization* process allows a node to catch up with a peer's chain. The process, handled by the [*SyncBlock*][sb] procedure, is triggered when receiving a block at a higher height than the $Tip$.

If the triggering block is a valid $Tip$'s successor, it is accepted as the new tip, and the SA loop ([*SALoop*][sl]) is restarted (to sync up the consensus protocol).

Instead, if the triggering block has height higher than the $Tip$'s successor, a protocol is initiated to request to the peer that sent the block all missing blocks. This protocol aims at making the node catch up with the chain of the chosen peer, referred to as *sync peer*.
The process consists in requesting the missing block to the sync peer and waiting for such blocks to be received. When received, each block, in the proper order, is verified and, if valid, accepted to the local chain.

More specifically, the protocol works as follows:
1. The node sends a $\mathsf{GetBlocks}$ message to $\mathcal{S}$ containing its local $Tip$;
2. The sync peer $\mathcal{S}$ replies with an $\mathsf{Inv}$ message containing the hashes of the blocks from the received $Tip$ and its current tip;
3. When the $\mathsf{Inv}$ message is received, the node requests unknown blocks with a $\mathsf{GetData}$ message;
4. The peer responds to the $\mathsf{GetData}$ message by sending all requested blocks, one by one, using $\mathsf{Block}$ messages.

During the protocol, if the sync peer sends a valid $Tip$'s successor, the node considers such peer as trustworthy and stops the SA loop ([*SALoop*][sl]) for efficiency purposes. 

Nonetheless, the sync peer is only given a limited amount of time (defined by [SyncTimeout][env]) to transmit blocks. If the timeout expires, the protocol is ended (see [*HandleSyncTimeout*][hst]) and consensus restarted. The timer to keep track of the timeout is set when starting the protocol, and reset each time a valid $Tip$'s successor is provided by the syncpeer.

### Procedures

#### *SyncBlock*
The *SyncBlock* procedure handles potential successors of the local $Tip$. The procedure accepts valid $Tip$'s successors and is responsible for initiating ([*StartSync*][ss]) and handling synchronization protocols run with nodes peers. Only one synchronization protocol can be run at a time, and only with a single peer (the *sync peer*).

When receiving blocks with higher height than the $Tip$'s successor, they are stored in the $BlockPool$. If receiving a valid $Tip$'s successor, this is accepted along with all valid successors in the $BlockPool$.

If an invalid $Tip$'s successor is received by a sync peer while running the protocol, the protocol is ended.

***Parameters***
- $\mathsf{B}$: received block
- $\mathcal{S}$: sender peer

***Algorithm***
<!-- inSync -->
1. If not synchronizing with any peer ($inSync = true$) :
   <!-- B > Tip+1 -->
   1. If $\mathsf{B}$ is higher than the $Tip$'s successor:
      1. Add $\mathsf{B}$ to the $BlockPool$
      2. Start synchronization protocol ([*StartSync*][ss]) with peer $\mathcal{S}$
   <!-- B = Tip+1 -->
   2. If $\mathsf{B}$ is the $Tip$'s successor:
      1. Verify $\mathsf{B}$ ([*VerifyBlock*][vb])
      2. If not valid failed, stop
      3. Accept $\mathsf{B}$ to the chain
      4. Propagate $\mathsf{B}$ to other peers
      5. Restart the consensus loop ([*SALoop*][sl])

<!-- outSync -->
1. If synchronizing with a peer ($inSync = false$) :
   <!-- B > Tip+1 -->
   1. If $\mathsf{B}$ is higher than the $Tip$'s successor :
      1. $BlockPool = BlockPool \cup \mathsf{B}$

   <!-- B = Tip+1 -->
   2. If $\mathsf{B}$ is the $Tip$'s successor :
      1. Verify $\mathsf{B}$ ([*VerifyBlock*][vb])
      2. If $\mathsf{B}$ is valid :
         1. Accept $\mathsf{B}$ to the chain ([*AcceptBlock*][ab])
         2. Accept all consecutive $\mathsf{B}$'s successors in $BlockPool$ ([*AcceptPoolBlocks*][apb])
         3. If $Tip$'height is $syncFrom$ :
            1. Stop the consensus loop ([*SALoop*][sl])
      3. If $\mathsf{B}$ is not valid :
         1. If $Tip$'height is $syncFrom$
         2. And the sender $\mathcal{S}$ is the sync peer ($syncPeer$)
            1. Stop syncing ($inSync = true$)

***Procedure***

$\textit{SyncBlock}(\mathsf{B}, \mathcal{S}):$
<!-- inSync -->
1. $\texttt{if } (inSync = true) :$
   <!-- B > Tip+1 -->
   1. $\texttt{if } (\mathsf{B}.Height > Tip.Height + 1) :$
      1. $BlockPool = BlockPool \cup \mathsf{B}$
      2. [*StartSync*][ss]$(\mathsf{B}, \mathcal{S})$
   <!-- B = Tip+1 -->
   2. $\texttt{if } (\mathsf{B}.Height = Tip.Height+1) :$
      1. $isValid$ = [*VerifyBlock*][vb]$(\mathsf{B})$
      2. $\texttt{if } (isValid = false): \texttt{stop}$
      3. [*AcceptBlock*][ab]$(\mathsf{B})$
      4. [*Propagate*][mx]$(\mathsf{B})$
      5. $\texttt{restart}$([*SALoop*][sl])

<!-- outSync -->
2. $\texttt{if } (inSync = false) :$
   <!-- B > Tip+1 -->
   1. $\texttt{if } (\mathsf{B}.Height > Tip.Height + 1) :$
      1. $BlockPool = BlockPool \cup \mathsf{B}$
   
   <!-- B = Tip+1 -->
   2. $\texttt{if } (\mathsf{B}.Height = Tip.Height+1) :$
      1. $isValid$ = [*VerifyBlock*][vb]$(\mathsf{B}, Tip)$
      2. $\texttt{if } (isValid = true):$
         1. [*AcceptBlock*][ab]$(\mathsf{B})$
         2. [*AcceptPoolBlocks*][apb]$()$
         3. $\texttt{if } (Tip.Height = syncFrom) :$
            1. $\texttt{stop}$([*SALoop*][sl])
      3. $\texttt{else}:$
         1. $\texttt{if } (Tip.Height = syncFrom)$
         2. $\texttt{and } (\mathcal{S} = syncPeer) :$
            1. $inSync = true$

<!-- TODO: in outSync, the condition (Tip.Height = syncFrom) could be removed: we stop SALoop only if active, and we stop syncing if S provides any invalid block at height tip+1 -->

#### *StartSync*
The *StartSync* procedure initiates the synchronization protocol with a sync peer ($\mathcal{S}$) on the base of a block $\mathsf{B}$ received from such peer.

***Parameters***
- $\mathsf{B}$: received block
- $\mathcal{S}$: sender peer

***Algorithm***
1. Set $syncFrom$ to $Tip$'s height
2. Set $syncTo$ to $\mathsf{B}$ (or $Tip$'s height plus $MaxSyncBlocks$)
3. Send a $\mathsf{GetBlocks}$ message to $\mathcal{S}$ to request missing blocks
4. Set $\mathcal{S}$ as the sync peer ($syncPeer$)
5. Start the sync timer
6. Start the sync timeout handler
7. Set as "syncing" ($inSync = false$)

***Procedure***

$\textit{StartSync}(\mathsf{B}, \mathcal{S}):$
1. $syncFrom = Tip.Height$
2. $syncTo = min(\mathsf{B}.Height, Tip.Height+MaxSyncBlocks)$
3. [*Send*][mx]$(\mathsf{GetBlocks}(Tip.Hash))$
4. $syncPeer = \mathcal{S}$
5. $\tau_{Sync} = \tau_{Now}$
6. [*HandleSyncTimeout*][hst]$()$
7. $inSync = false$

#### *HandleSyncTimeout*
The *HandleSyncTimeout* procedure is executed when the node is running a synchronization protocol with a peer and the corresponding sync timeout expires. When this occurs, the sync protocol is ended and the SA loop restarted, if needed.

***Procedure***
$\textit{HandleSyncTimeout}():$
- $\texttt{loop}:$
1. $\texttt{if }(inSync = false):$
   1. $\texttt{if }(\tau_{Now} \gt \tau_{Sync} + SyncTimeout):$
      1. $inSync = true$
      2. $\texttt{if } (\texttt{running}$([*SALoop*][sl]$) = false)$:
        1. $\texttt{start}$([*SALoop*][sl])
2. $\texttt{else}: \texttt{stop}$


#### *AcceptPoolBlocks*
The *AcceptPoolBlocks* procedure accepts all successive blocks in $BlockPool$ from height $Tip.Height+1$ until no successor is available. It is called after receiving a valid block at height $Tip.Height+1$ (which updates the $Tip$) to accept previously-collected blocks.

***Algorithm***
1. Get all successive blocks in $BlockPool$ starting from $Tip$'s successor
2. For each retrieved block $\mathsf{B}_i$:
   1. Verify $\mathsf{B}_i$ ([*VerifyBlock*][vb])
      1. If verification fails, stop
   2. Accept $\mathsf{B}_i$ to the chain
   3. Reset sync timer $\tau_{Sync}$
   4. If $\mathsf{B}_i$'s height is $syncTo$
      1. Stop syncing ($inSync = true$)
      2. Restart consensus loop ([*SALoop*][sl])

***Procedure***

$\textit{AcceptPoolBlocks}():$
1. $\boldsymbol{Successors} =$ *getFrom*$(BlockPool, \mathsf{B}_{Tip.Height+1})$
2. $\texttt{for } i = 0 \dots |\boldsymbol{Successors}|{-}1 :$
   1. $isValid$ = [*VerifyBlock*][vb]$(\mathsf{B}_i, Tip)$
      1. $\texttt{if } (isValid = false): \texttt{stop}$
   2. [*AcceptBlock*][ab]$(\mathsf{B}_i)$
   3. $\tau_{Sync} = \tau_{Now}$
   4. $\texttt{if } (\mathsf{B}_i.Height = syncTo):$
      1. $inSync = true$
      2. $\texttt{start}$([*SALoop*][sl])


<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md -->
[env]: #environment
[ab]:  #acceptblock
[apb]: #acceptpoolblocks
[cs]:  #consensus-state
[fin]: #finality
[rf]:  #rolling-finality
[crf]: #checkrollingfinality
[fal]: #fallback
[hst]: #handlesynctimeout
[hb]:  #HandleBlock
[syn]: #synchronization
[sb]:  #syncblock
[ss]:  #startsync
[vb]:  #verifyblock
[vbh]: #verifyblockheader
[vc]:  #verifycertificate
[vv]:  #verifyvotes
[hq]:  #HandleQuorum
[mw]:  #makewinning
[va]:  #VerifyQuorum

<!-- Blockchain -->
[b]:  https://github.com/dusk-network/dusk-protocol/tree/main/blockchain/README.md#block
[lc]: https://github.com/dusk-network/dusk-protocol/tree/main/blockchain/README.md#chain

<!-- Basics -->
[sv]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#stepvotes
[certs]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#certificates
[sc]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#subcommittee
[cc]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#countcredits
[ec]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#ExtractCommittee

<!-- Consensus -->
[cenv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#environment
[sa]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#overview
[sl]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#saloop
[sai]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#saiteration
[gq]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#GetQuorum
[gsn]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#GetStepNum

[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md

<!-- Sortition -->
[ds]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md
[dsp]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#deterministic-sortition-ds 

<!-- Messages -->
[mx]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#procedures
[bmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#block
[gcmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#getcandidate
[ms]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#signatures

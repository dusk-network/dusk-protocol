# Chain Management
*Chain management* refers to how the SA protocol handles the local chain with respect to consensus-produced winning blocks, fork and out-of-sync events.

In particular, this section describes how new blocks are accepted to the local blockchain and how this chain can be updated when receiving valid blocks from the network.

### ToC
- [Overview](#overview)
  - [Block Finality](#block-finality)
- [Environment](#environment)
- [Block Verification](#block-verification)
  - [*VerifyBlock*](#verifyblock)
  - [*VerifyCertificate*](#verifycertificate)
  - [*VerifyBlockHeader*](#verifyblockheader)
- [Block Management](#block-management)
  - [*AcceptBlock*](#acceptblock)
  - [*ProcessBlock*](#processblock)
- [Fallback](#fallback)
  - [*Fallback* Procedure](#fallback-procedure)
- [Synchronization](#synchronization)
  - [*SyncBlock*](#syncblock)
  - [*StartSync*](#startsync)
  - [*HandleSyncTimeout*](#handlesynctimeout)
  - [*AcceptPoolBlocks*](#acceptpoolblocks)

## Overview
At node level, there are two ways for blocks to be added to the local chain. The first one is through the consensus protocol, that is, at the end of a successful round. In this case, the candidate block, for which a quorum has been reached in both Reduction phases, becomes a *winning block* and is therefore added to the chain as the new tip, updating the state accordingly (see [*AcceptBlock*][ab]).
At the same time, it is possible that a block is received from the network, which has already reached consensus (proved by the *block certificate*). When the received block is not in the local chain, it can indicate two possible situations:
 
 - *out-of-sync*: the node fell behind the main chain, that is, its $Tip$ is part of the main chain but is at a lower height then the main chain's tip; in this case, the node needs to retrieve missing blocks to catch up with the main chain. This process is called [*synchronization*][syn] and is run with the peer that sent the triggering block.

 - *fork*: two or more blocks have reached consensus at the same height (but different iteration); in this case, the node must first decide whether to stick to the local chain or switch to the other one. To switch, the node runs a [*fallback*][fal] process that reverts the local chain (and state) to the last finalized block and then starts the synchronization procedure to catch up with the main chain.

Incoming blocks (transmitted via [Block][bmsg] messages) are handled by the [*ProcessBlock*][pb] procedure, which leverages the [*Fallback*][fal] and [*SyncBlock*][sb] procedures to manage forks and out-of-sync cases, respectively.

## Certificate
The $\mathsf{Certificate}$ structure contains the [Reduction][rmsg] votes of the [quorum committee][sc] that reached consensus on a candidate block.
It is composed of two $\mathsf{StepVotes}$ structures, one for each [Reduction][red] step.


| Field             | Type            | Size     | Description                          |
|-------------------|-----------------|----------|--------------------------------------|
| $FirstReduction$  | [StepVotes][sv] | 56 bytes | Aggregated votes of first reduction  |
| $SecondReduction$ | [StepVotes][sv] | 56 bytes | Aggregated votes of second reduction |

The $\mathsf{Certificate}$ structure has a total size of 112 bytes.

### Block Finality
Due to the asynchronous nature of the network, more than one block can reach consensus in the same round (but in different iterations). When this occurs, some nodes will have a different block for the same height, creating a *fork*. This is typically due to consensus messages being delayed or lost due to network congestion.

When a fork is detected, nodes automatically switch to the block that reached quorum at a lower iteration. Consequently, it is possible for a block to be reverted and replaced by another block (see [*Fallback*][fal]). At the same time, blocks reaching consensus at iteration 1 can't be replaced by lower-iteration ones. We then call such blocks *final*.

Given the above, at any moment, the local chain can be considered as made of two parts: a final one, which is from the genesis to the last finalized block, and the non-final one, which includes all blocks after the last final block. Blocks in the non-final part can potentially be reverted until a new final block is added. In contrast, the final part cannot be reverted in any way and is the definitive. When the chain tip is final, then the whole chain is final.

**Last Final Block**
Due to its relevance, we formally define the *last final block* as the highest block that has been marked as final, and denote it with $\mathsf{B}^f$.

### Instant Finality
<!-- DOING -->

### Rolling Finality
<!-- DOING -->

## Environment
The environment for the block-processing procedures include node-level parameters, influencing the node's behavior during synchronization, and state variables that help keep track of known blocks and handle the synchronization protocol execution.

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
  - $\mathsf{C}_\mathsf{B^p} = \mathsf{B}.PrevBlockCertificate$
  - $\mathsf{C}_\mathsf{B} = \mathsf{B}.Certificate$
1. $isValid$ = [*VerifyBlockHeader*][vbh]$(\mathsf{B}^p,\mathsf{B})$
2. $\texttt{if } (isValid = false): \texttt{output } false$
3. $isValid$ = [*VerifyCertificate*][vc]$(\mathsf{C}_\mathsf{B^p},\eta_{\mathsf{B}^p},r_{\mathsf{B}^p},s_{\mathsf{B}^p})$
4. $\texttt{if } (isValid = false): \texttt{output } false$
5. $isValid$ = [*VerifyCertificate*][vc]$(\mathsf{C}_\mathsf{B},\eta_{\mathsf{B}^p},r_{\mathsf{B}},s_{\mathsf{B}})$
6. $\texttt{if } (isValid = false): \texttt{output } false$
7. $\texttt{for } \mathsf{C}_i \texttt{ in } \mathsf{B}.FailedIterations$
   1. $isValid =$ [*VerifyCertificate*][vc]$(\mathsf{C}_i)$
   2. $\texttt{if } (isValid = false): \texttt{output } false$
8.  $\texttt{output } true$

### VerifyBlockHeader
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


### VerifyCertificate
*VerifyCertificate* checks a block's certificate by verifying the two Reduction aggregated signatures against the respective committees.

***Parameters***
- $\mathsf{C}$: the block's certificate
- $b$: block's hash
- $r$: round
- $s$: step

***Algorithm***
1. Check both Reduction votes are present
2. Verify First-Reduction votes
   1. If votes are not valid, output $false$
3. Verify Second-Reduction votes
   1. If votes are not valid, output $false$
4. Output $true$

***Procedure***

$\textit{VerifyCertificate}(\mathsf{C}, b, r, i):$
- $\texttt{set}:$
   - $`\mathsf{V}_1, \mathsf{V}_2 \leftarrow \mathsf{C}`$
   - $s_1 = i \times 3 + 1$
   - $s_2 = i \times 3 + 2$
1. $\texttt{if } (\mathsf{V}_1 = NIL) \texttt{ or } (\mathsf{V}_2 = NIL):$
   1. $\texttt{output } false$
2. $r_1 =$ [*VerifyVotes*][vv]$`(\mathsf{V}_1, b, r, s_1)`$
3. $r_2 =$ [*VerifyVotes*][vv]$`(\mathsf{V}_2, b, r, s_2)`$
4. $\texttt{if } (r_1{=}true) \texttt{ and } (r_2{=}true) :$
   1. $\texttt{output } true$
5. $\texttt{else}:$ 
   1. $\texttt{output } false$

### VerifyVotes
$VerifyVotes$ checks the aggregated votes for a candidate are valid and reach the quorum.

***Parameters***
- $\mathsf{V}$: $\mathsf{StepVotes}$ with the aggregated votes
- $b$: the block's hash
- $r$: round
- $s$: step

***Algorithm***
1. Compute subcommittee $C^{\boldsymbol{bs}}$ from $\mathsf{V}.BitSet$
2. If $b$ is a timeout vote ($NIL$)
   1. Set quorum target $q$ to $NilQuorum$
3. Otherwise, set $q$ to $Quorum$
4. If credits in $C^{\boldsymbol{bs}}$ are less than $q$
   1. Output $false$
5. Aggregate public keys of $C^{\boldsymbol{bs}}$ members
6. Compute hash $\eta$ of round $r$, step $s$, and block $b$
7. Verify aggregated signature over $\eta$

***Procedure***

$VerifyVotes(\mathsf{V}, b, r, s)$:
- $\texttt{set}:$
  - $\boldsymbol{bs}, \sigma_{\boldsymbol{bs}} \leftarrow \mathsf{V}$
1. $C^{\boldsymbol{bs}} = $ [SubCommittee][sc]$(C_{r}^{s}, \boldsymbol{bs})$
2. $\texttt{if } (b = NIL):$
   1. $\texttt{set } q = NilQuorum$
3. $\texttt{else}: \texttt{set } q = Quorum$
4. $\texttt{if } ($[*CountCredits*][cc]$(C_{r}^{s}, C^{\boldsymbol{bs}}) \lt q):$
   1. $\texttt{output } false$
5. $pk_{\boldsymbol{bs}} = AggregatePKs(C^{\boldsymbol{bs}})$
6. $\eta = Hash_{Blake2B}(r||s||hash)$
7. $\texttt{output } Verify_{BLS}(\eta, pk_{\boldsymbol{bs}}, \sigma_{\boldsymbol{bs}})$


## Block Management
We here define block-management procedures: 
  - $MakeWinning$: sets a candidate block as the winning block of the round
  - $AcceptBlock$: accept a block as the new tip
  - $ProcessBlock$: process a block from the network

### MakeWinning
*MakeWinning* adds a certificate to a candidate block and set the winning block variable $\mathsf{B}^w$.

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


### AcceptBlock
*AcceptBlock* sets a block $\mathsf{B}$ as the new chain $Tip$. It also updates the local state accordingly by executing all transactions in the block and setting the $Provisioners$ state variable.
If consensus was reached in the first iteration, the block is marked as *final*.

***Parameters***
- $\mathsf{B}$: the block to accept as the new chain tip

***Algorithm***
1. Extract $Transactions$, $GasLimit$, and $Generator$ from block $\mathsf{B}$
2. Generate new state ($newState$) by applying $Transactions$ on the current $State$, and assigning the block reward to $Generator$
3. Update $Provisioners$ set
4. If $Iteration$ is 1, make the block final
5. Set $Tip$ to block $\mathsf{B}$

***Procedure***

$\textit{AcceptBlock}(\mathsf{B}):$
1. $\texttt{set }$:
   - $\boldsymbol{txs} = \mathsf{B}.Transactions$
   - $gas = \mathsf{B}.GasLimit$
   - $pk_{\mathcal{G}} = \mathsf{B}.Generator$
2. $newState =$ *ExecuteTransactions*$(State, \boldsymbol{txs}, gas, pk_{\mathcal{G}})$
3. $Provisioners = newState.Provisioners$
4. $\texttt{if } (Iteration = 1):$ *MakeFinal*$(\mathsf{B})$
5. $Tip = \mathsf{B}$


### ProcessBlock
The *ProcessBlock* procedure processes a full block received from the network and decides whether to trigger the synchronization or fallback procedures.
The procedure acts depending on the block's height: if the block has the same height as the $Tip$, but lower $Iteration$, it starts the [*Fallback*][fal] procedure; if the block's height is more than $Tip.Height+1$, it executes the [*SyncBlock*][sb] is executed to start or continue the synchronization process; if the block as height lower than the $Tip$, the block is discarded.

***Parameters*** 
- $\mathsf{M}^{Block}$: the incoming $\mathsf{Block}$ message

***Algorithm***
- Extract the block $\mathsf{B}$ and the message sender $\mathcal{S}$
1. Check block hash $\mathsf{H}_\mathsf{B}$
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

$\textit{ProcessBlock}(\mathsf{M}^{Block}):$
- $\mathsf{B},\mathcal{S} \leftarrow \mathsf{M}^{Block}$
1. $isValidHash = (\mathsf{B}.Hash =$ *Hash*$`_{SHA3}(\mathsf{H}_{\mathsf{B}}))`$ \
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

<!-- TODO: Define *MakeFinal* -->

## Fallback
The *Fallback* procedure reverts the local state to the last [finalized][fin] block. The procedure is triggered by [*ProcessBlock*][pb] when receiving a block at the same height as the $Tip$ but with lower $Iteration$. 

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


### SyncBlock
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

### StartSync
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

### HandleSyncTimeout
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


### AcceptPoolBlocks
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
2. $n = \textit{len}(\boldsymbol{Successors}) - 1$ \
   $\texttt{for } i = 0 \dots n :$
   1. $isValid$ = [*VerifyBlock*][vb]$(\mathsf{B}_i, Tip)$
      1. $\texttt{if } (isValid = false): \texttt{stop}$
   2. [*AcceptBlock*][ab]$(\mathsf{B}_i)$
   3. $\tau_{Sync} = \tau_{Now}$
   4. $\texttt{if } (\mathsf{B}_i.Height = syncTo):$
      1. $inSync = true$
      2. $\texttt{start}$([*SALoop*][sl])


<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#block-finality -->
[env]:  #environment
[ab]:   #acceptblock
[apb]:  #acceptpoolblocks
[bf]:   #block-finality
[cert]: #certificate
[fal]:  #fallback
[hst]:  #handlesynctimeout
[pb]:   #processblock
[syn]:  #synchronization
[sb]:   #syncblock
[ss]:   #startsync
[vb]:   #verifyblock
[vc]:   #verifycertificate
[vv]:   #verifyvotes

<!-- Blockchain -->
[b]:   https://github.com/dusk-network/dusk-protocol/tree/main/blockchain/README.md#block
<!-- Consensus -->
[fin]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#finality
[sl]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#saloop
[vbh]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#verifyblockheader
[vc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#verifycertificate
<!-- Reduction -->
[red]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md
[sv]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md#stepvotes
<!-- Messages -->
[mx]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-exchange
[bmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#block-message
[rmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#reduction-message
<!-- Sortition -->
[sc]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#subcommittee
[cc]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#countcredits
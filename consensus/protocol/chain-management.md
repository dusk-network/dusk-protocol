# Chain Management
This section describes how new blocks are accepted into the local blockchain and how this chain can be updated when receiving valid blocks from the network.

**ToC**
  - [Block Verification](#block-verification)
    - [Procedures](#procedures)
      - [*VerifyBlock*](#verifyblock)
      - [*VerifyBlockHeader*](#verifyblockheader)
      - [*VerifyAttestation*](#verifyattestation)
      - [*VerifyVotes*](#verifyvotes)
  - [Block Handling](#block-handling)
    - [Environment](#environment)
    - [Procedures](#procedures-1)
      - [*HandleBlock*](#handleblock)
      - [*HandleQuorum*](#handlequorum)
      - [*MakeWinning*](#makewinning)
      - [*AcceptBlock*](#acceptblock)
      - [*Fallback*](#fallback)
  - [Synchronization](#synchronization)
    - [Environment](#environment-1)
    - [Procedures](#procedures-2)
      - [*SyncBlock*](#syncblock)
      - [*PreSync*](#presync)
      - [*StartSync*](#startsync)
      - [*HandleSyncTimeout*](#handlesynctimeout)
      - [*AcceptPoolBlocks*](#acceptpoolblocks)


## Block Verification
We here define the procedures to verify the validity of a block: [*VerifyBlock*][vb], [*VerifyBlockHeader*][vbh], [*VerifyAttestation*][va], and [*VerifyVotes*][vv].

### Procedures
#### *VerifyBlock*
This procedure verifies a block is a valid successor of another block $\mathsf{B}^p$ (commonly, the $Tip$) and contains a valid Attestation. If both conditions are met, it returns $true$, otherwise, it returns $false$.

If the block is an [Emergency Block][eb], the block $Attestation$ and $FailedIterations$ are not verified.

**Parameters**
- $\mathsf{B}$: the block to verify
- $\mathsf{B}^p$: the alleged $\mathsf{B}$'s parent

**Algorithm**
1. Verify $\mathsf{B}$'s header ([*VerifyBlockHeader*][vbh])
2. If header's not valid, output $false$
3. Verify $\mathsf{B}^p$'s certificate $\mathsf{A}_\mathsf{B^p}$ ([*VerifyAttestation*][va])
4. If $\mathsf{A}^p$ is not valid, output $false$
5. If $\mathsf{B}$ is an Emergency Block
   1. Output $true$
6. Verify $\mathsf{B}$'s attestation $\mathsf{A_B}$ ([*VerifyAttestation*][va])
7. If attestation is not valid, output $false$
8. For each attestation $\mathsf{A}_i$ in $FailedIterations$
   1. Verify $\mathsf{A}_i$ ([*VerifyAttestation*][va])
   2. If $\mathsf{A}_i$ is not valid, output $false$
9.  If all verifications succeeded, output $true$

**Procedure**
$\textit{VerifyBlock}(\mathsf{B}):$
- $\textit{set }:$
  - $\mathsf{A}_{\mathsf{B}^p} = \mathsf{B}.PrevBlockCertificate$
  - $\mathsf{A_B} = \mathsf{B}.Attestation$
  - $\upsilon_\mathsf{B} = (\mathsf{B}.PrevBlock,\mathsf{B}.Round,\mathsf{B}.Iteration,Valid,\eta_\mathsf{B})$
  - $\upsilon_{\mathsf{B}^p} = (\mathsf{B}^p.PrevBlock,\mathsf{B}^p.Round,\mathsf{B}^p.Iteration,Valid,\eta_{\mathsf{B}^p})$
1. $isValid$ = [*VerifyBlockHeader*][vbh]$(\mathsf{B}^p,\mathsf{B})$
2. $\texttt{if } (isValid = false): \texttt{output } false$
3. $isValid$ = [*VerifyAttestation*][va]$`(\mathsf{A}_{\mathsf{B}^p},\upsilon_\mathsf{B})`$
4. $\texttt{if } (isValid = false): \texttt{output } false$
5. $\texttt{if } ($[*isEmergencyBlock*][ieb]$(\mathsf{B}) = true):$
   1. $\texttt{output } true$
6. $isValid$ = [*VerifyAttestation*][va]$`(\mathsf{A}_{\mathsf{B}},\upsilon_{\mathsf{B}^p})`$
7. $\texttt{if } (isValid = false): \texttt{output } false$
8. $\texttt{for } i = 0 \dots |\mathsf{B}.FailedIterations|$
   - $\mathsf{A}_i = \mathsf{B}.FailedIterations[i]$
   1. $\texttt{if } (\mathsf{A}_i \ne NIL) :$
      <!-- TODO: support Invalid/NoQuorum votes -->
      - $\upsilon_i = (\mathsf{B}.PrevBlock,\mathsf{B}.Round,i,NoCandidate)$[^1]
      1. $isValid =$ [*VerifyAttestation*][va]$(\mathsf{A}_i, \upsilon_i)$
      2. $\texttt{if } (isValid = false): \texttt{output } false$
9.  $\texttt{output } true$

#### *VerifyBlockHeader*
This procedure returns $true$ if all block header fields are valid with respect to the previous block and the included transactions. If so, it outputs $true$, otherwise, it outputs $false$.

**Parameters**
- $\mathsf{B}$: block to verify
- $\mathsf{B}^p$: previous block

**Algorithm**
1. Check $Version$ is $0$
2. Check $Hash$ is the header's hash
3. Check $Height$ is $\mathsf{B}^p$'s height plus 1
4. Check $PrevBlock$ is $\mathsf{B}^p$'s hash
5. Check $Seed$ is the generator's signature of the previous seed
6. Check transaction root is correct with respect to the transaction set
7. Check state hash corresponds to the result of the state transition over $\mathsf{B}^p$
- If any check failed
  1. Output $false$
- Otherwise, output $true$

**Procedure**

$\textit{VerifyBlockHeader}(\mathsf{B}, \mathsf{B}^p)$:
- $newState =$ *ExecuteTransactions*$(State_{\mathsf{B}^p}, \mathsf{B}.Transactions, BlockGas, pk_{G_\mathsf{B}})$
- $\texttt{if }$
  1. $(\mathsf{B}.Version > 0)$ 
  2. $\texttt{or } (\mathsf{B}.Hash \ne \eta(\mathsf{B}))$
  3. $\texttt{or } (\mathsf{B}.Height \ne \mathsf{B}^p.Height)$
  4. $\texttt{or } (\mathsf{B}.PreviousBlock \ne \mathsf{B}^p.Hash)$
  5. $\texttt{or } (\mathsf{B}.Seed \ne $*Sign*$(\mathsf{B}.Generator, \mathsf{B}^p.Seed))$
  6. $\texttt{or } (\mathsf{B}.TransactionRoot \ne MerkleTree(\mathsf{B}.Transactions).Root)$
  7. $\texttt{or } (\mathsf{B}.StateRoot \ne MerkleTree(newState).Root):$
     1. $\texttt{output } false$

  8. $\texttt{output } true$


#### *VerifyAttestation*
This procedure checks a block's Attestation by verifying the [Validation][val] and [Ratification][rat] aggregated signatures against the respective committees.

**Parameters**
- $\mathsf{A}$: the attestation to verify
- $\upsilon$: the vote's data, containing: the previous block's hash, the iteration number, the winning vote, the block's hash

**Algorithm**
1. Check both Validation and Ratification votes are present
2. Verify Validation votes
3. If votes are not valid, output $false$
4. Verify Ratification votes
5. If votes are not valid, output $false$
6. Output $true$

**Procedure**

$\textit{VerifyAttestation}(\mathsf{A}, \upsilon):$
- $\texttt{set}:$
   - $`\mathsf{SV}^V, \mathsf{SV}^R \leftarrow \mathsf{A}`$
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

#### *VerifyVotes*
This procedure checks the aggregated votes are valid and reach the target quorum.

**Parameters**
- $\mathsf{SV}$: $\mathsf{StepVotes}$ with the aggregated votes
- $\upsilon$: the [signature value][ms]
- $Q$: the target quorum
- $\mathcal{C}$: the step committee

**Algorithm**
1. Compute subcommittee $C^{\boldsymbol{bs}}$ from $\mathsf{SV}.BitSet$
2. If credits in $C^{\boldsymbol{bs}}$ are less than the target quorum $Q$
   1. Output $false$
3. Aggregate public keys of $C^{\boldsymbol{bs}}$ members
4. Verify aggregated signature over $\upsilon$

**Procedure**

$\textit{VerifyVotes}(\mathsf{SV}, \upsilon, Q)$:
- $\texttt{set}:$
  - $\boldsymbol{bs}, \sigma_{\boldsymbol{bs}} \leftarrow \mathsf{SV}$
1. $\mathcal{C}^{\boldsymbol{bs}}=$ [*SubCommittee*][sc]$(\mathcal{C}, \boldsymbol{bs})$
2. $\texttt{if } ($[*CountCredits*][cc]$(\mathcal{C}, \boldsymbol{bs}) \lt Q):$
   1. $\texttt{output } false$
3. $pk_{\boldsymbol{bs}} = AggregatePKs(C^{\boldsymbol{bs}})$
4. $\texttt{output } Verify_{BLS}(\upsilon, pk_{\boldsymbol{bs}}, \sigma_{\boldsymbol{bs}})$

<p><br></p>

## Block Handling
At node level, there are two ways for blocks to be added to the [local chain][lc]: being the winning candidate of a [consensus round][sa] or being an attested block from the network. In the first case, the candidate block, for which a quorum has been reached in both [Validation][val] and [Ratification][rat] steps, becomes a *winning block* and is therefore added to the chain as the new tip (see [*AcceptBlock*][ab]).
In the latter case, a block is received from the network, which has already reached consensus (proved by the block [Attestation][atts]). 

There are two main reasons a network block is not in the chain:
 
 - *out-of-sync*: the node fell behind the main chain, that is, its $Tip$ is part of the main chain but is at a lower height than the main chain's tip; in this case, the node needs to retrieve missing blocks to catch up with the network. This process is called [*synchronization*][syn] and is run with the peer that sent the triggering block.

 - *fork*: two or more candidates reached consensus in the same round (but different iteration); in this case, the node must first decide whether to stick to the local chain or switch to the other one. To switch, the node runs a [*fallback*][fal] process that reverts the local chain (and state) to the last finalized block and then starts the synchronization procedure to catch up with the main chain.

Incoming blocks (transmitted via [Block][bmsg] messages) are handled by the [*HandleBlock*][hb] procedure, which can trigger the [*Fallback*][fal] and [*SyncBlock*][sb] procedures to manage forks and out-of-sync cases, respectively.


### Environment

| Name          | Type                 | Description                |
|---------------|----------------------|----------------------------|
| $Blacklist$   | $\mathsf{Block}$ [ ] | list of blacklisted blocks |


### Procedures
We define the following block-handling procedures: 
  - [*HandleBlock*][hb]: handle `Block` messages from the network
  - [*HandleQuorum*][hq]: handle `Quorum` messages from the network
  - [*MakeWinning*][mw]: sets a candidate block as the winning block of the round
  - [*AcceptBlock*][ab]: accept a block as the new tip
  - [*Fallback*][fal]: reverts the chain to a specific block

#### *HandleBlock*
This procedure processes a full block received from the network and decides whether to trigger the synchronization or fallback procedures.
The procedure acts depending on the block's height: if the block has the same height as a local chain block, but it has lower $Iteration$, it starts the [*Fallback*][fal] procedure; if the block's height is more than $Tip.Height+1$, it executes the [*SyncBlock*][sb] is executed to start or continue the synchronization process.

**Parameters** 
- $\mathsf{M}^{Block}$: the incoming $\mathsf{Block}$ message

**Algorithm**
1. Loop:
   1. If a $\mathsf{Block}$ message $\mathsf{M^B}$ is received:
      - Extract the block $\mathsf{B}$ and the message sender $\mathcal{S}$
      1. Check $\mathsf{B}$ hash is valid
      2. Check $\mathsf{B}$ is not blacklisted
      3. If $\mathsf{B}$ is higher than $Tip$:
         1. Start *synchronization* ([*SyncBlock*][sb])
      4. Otherwise, if $\mathsf{B}$ is higher than the last final block $\mathsf{B}^f$:
         1. Check if $\mathsf{B}$ is not already in our chain
         2. If $\mathsf{B}$'s parent is in our chain
         3. And its local sibling has a higher iteration number
            1. Check $\mathsf{B}$'s validity
            2. If $\mathsf{B}$ is valid:
               1. Stop consensus loop [*SALoop*][sl]
               2. Revert chain to $\mathsf{B}$'s parent ([*Fallback*][fal])
               3. Accept $\mathsf{B}$ to the chain
               4. Restart the consensus loop [*SALoop*][sl]

**Procedure**

$\textit{HandleBlock}():$
1. $\texttt{loop}$:   
   1.  $\texttt{if } (\mathsf{M^B} =$ [*Receive*][mx]$(\mathsf{Block}) \ne NIL):$
       - $\texttt{set}:$
        - $`\mathsf{B} \leftarrow \mathsf{M^B}`$
        - $\eta^p = \mathsf{B}.PrevBlockHash$
       1. $\texttt{if } (\mathsf{B}.Hash \ne \eta(\mathsf{B})): \texttt{break}`$
       2. $\texttt{if } (\mathsf{B} \in Blacklist) : \texttt{break}$
          <!-- B.Height > Tip.Height -->
       3. $\texttt{if } (\mathsf{B}.Height > Tip.Height) :$
          1. [*SyncBlock*][sb]$(\mathsf{M^B})$
          <!-- B.Height <= Tip.Height -->
       4. $\texttt{else if } (\mathsf{B}.Height > \mathsf{B}^f.Height) :$
          1. $\texttt{if } (\mathsf{B} \in \textbf{Chain}):  \texttt{break}$
          2. $\texttt{if } (\eta^p \in \textbf{Chain})$
          3. $\texttt{and } (\mathsf{B}.Iteration \lt \textbf{Chain}[\mathsf{B}.Height].Iteration):$
             1. $isValid =$ [*VerifyBlock*][vb]$(\mathsf{B}, \mathsf{B}_{\eta^p})$
             2. $\texttt{if } (isValid = true)$
                1. $\texttt{stop}($[*SALoop*][sl]$)$
                2. [*Fallback*][fal]$(\eta^p)$
                3. [*AcceptBlock*][ab]$(\mathsf{B})$
                4. $\texttt{start}($[*SALoop*][sl]$)$

#### *HandleQuorum*
This procedure manages `Quorum` messages for the current round $R$. If the message was received from the network, it is first verified ([*VerifyQuorum*][vq]) and then propagated.
The corresponding candidate block is then marked as the winning block of the round ([*MakeWinning*][mw]).

**Parameters**
- [SA Environment][cenv]
- $R$: round number

**Algorithm**
1. Loop:
   1. If a $\mathsf{Quorum}$ message $\mathsf{M}^Q$ is received for round $R$:
      1. Verify $\mathsf{M}^Q.Attestation$ ($\mathsf{A}$) is valid ([*VerifyQuorum*][vq])
      2. If valid:
         1. Propagate $\mathsf{M}^Q$
         2. If the quorum vote is $Valid$
            1. Fetch candidate $\mathsf{B}^c$ from $\mathsf{M}^Q.BlockHash$
            2. If $\mathsf{B}^c$ is unknown, request it to peers ([*GetCandidate*][gcmsg])
            3. Set the winning block $\mathsf{B}^w$ to $\mathsf{B}^c$ with $\mathsf{M}^Q.Attestation$
         3. Otherwise
            1. Add $\mathsf{A}$ to the $\boldsymbol{FailedAttestations}$ list
         4. Stop [*SAIteration*][sai]

**Procedure**
<!-- TODO: define candidate pool/db and functions, eg FetchCandidate -->
$\textit{HandleQuorum}( R ):$
1. $\texttt{loop}$:   
   1.  $\texttt{if } (\mathsf{M}^Q =$ [*Receive*][mx]$(\mathsf{Quorum}, R) \ne NIL):$
       -  $\texttt{set}:$
          - $`\mathsf{CI}, \mathsf{VI}, \mathsf{A} \leftarrow \mathsf{M}^Q`$
          - $`\eta_{\mathsf{B}^p}, R_{\mathsf{M}}, I_{\mathsf{M}}, \leftarrow \mathsf{CI}`$
          - $`v, \eta_{\mathsf{B}^c} \leftarrow \mathsf{VI}`$
          - $\upsilon = (\eta_{\mathsf{B}^p}||I_{\mathsf{M}}||v||\eta_\mathsf{B})$

       1. $isValid =$ [*VerifyAttestation*][va]$(\mathsf{A}, \upsilon)$
       2. $\texttt{if } (isValid = true) :$
          1. $\texttt{stop}$([*SAIteration*][sai])
          2. [*Propagate*][mx]$(\mathsf{M}^Q)$
          3. $\texttt{if } (v = Valid) :$
             1. $\mathsf{B}^c =$ *FetchCandidate* $(\eta_{\mathsf{B}^c})$
             2. $\texttt{if } (\mathsf{B}^c = NIL) :$
               - $\mathsf{B}^c =$ [*GetCandidate*][gcmsg]$(\eta_{\mathsf{B}^c})$
             3. [*MakeWinning*][mw]$(\mathsf{B}^c, \mathsf{A})$
          4. $\texttt{else } :$
             1. $\boldsymbol{FailedAttestations}[I_{\mathsf{M}}] = {\mathsf{A}}$


#### *MakeWinning*
This procedure adds an attestation to a candidate block and sets the winning block variable $\mathsf{B}^w$.

**Parameters**
- $\mathsf{B}$: candidate block
- $\mathsf{A}$: block attestation

**Algorithm**
1. Add attestation $\mathsf{A}$ to candidate block $\mathsf{B}$
2. Set winning block $\mathsf{B}^w$ to $\mathsf{B}$

**Procedure**

$\textit{MakeWinning}(\mathsf{B}, \mathsf{A}):$
1. $\mathsf{B}.Attestation = \mathsf{A}$
2. $\mathsf{B}^w = \mathsf{B}$


#### *AcceptBlock*
This procedure sets a block $\mathsf{B}$ as the new chain $Tip$. It also updates the local state accordingly by executing all transactions in the block and setting the $Provisioners$ state variable. 

**Parameters**
- $\mathsf{B}$: the block to accept as the new chain tip

**Algorithm**
1. Extract $Transactions$, $GasLimit$, and $Generator$ from block $\mathsf{B}$
2. Update $SystemState$ by executing $Transactions$ and assigning the block [rewards and penalties][inc]
3. Update the $Provisioners$ set
4. Set $Tip$ to block $\mathsf{B}$
5. Compute the consensus state $s$ of $\mathsf{B}$
6. Add $(Tip, s)$ to the local chain
7. Check Rolling Finality

**Procedure**

$\textit{AcceptBlock}(\mathsf{B}):$
1. $\texttt{set }$:
   - $\boldsymbol{txs} = \mathsf{B}.Transactions$
   - $gas = \mathsf{B}.GasLimit$
   - $pk_{\mathcal{G}} = \mathsf{B}.Generator$
   - $h = \mathsf{H_B}.Height$
2. $SystemState =$ [*ExecuteTransactions*][xt]$(SystemState, \boldsymbol{txs}, gas, pk_{\mathcal{G}})$
3. $Provisioners = SystemState.Provisioners$
4. $Tip = \mathsf{B}$
5. $s =$ *GetBlockState*$(\mathsf{B})$
6. $\textbf{Chain}[h]=(\mathsf{B}, s)$
7. [*CheckRollingFinality*][crf]$()$



#### *Fallback*
This procedure takes the hash $\eta$ of a block in the local chain and deletes all blocks from $Tip$ to $\mathsf{B}_\eta$ excluded. 
It is triggered by [*HandleBlock*][hb] when receiving a block with height equal or lower than the local $Tip$ and with lower $Iteration$. 

This event indicates that a quorum was reached on a previous candidate and that the node is currently on a fork. To guarantee nodes converge on a common block, we give priority to blocks produced at lower iterations.

<!-- TODO: Is it safe to blacklist these blocks? Is it possible that we accept this block while the rest of the network moves on the delete branch?-->
Deleted blocks are blacklisted (because they belong to a lower-priority fork) and their transactions are pushed back to the mempool. The local $SystemState$ and $Provisioners$ are updated accordingly.

**Parameters**
- $\eta$: the hash of the block to which to revert

**Algorithm**
1. For each block $`\mathsf{B}_i`$ between $Tip$ and $\mathsf{B}_\eta$
   1. Push $\mathsf{B}_i$'s transactions back to mempool
   2. Delete $\mathsf{B}_i$ from the chain
   3. Blacklist $\mathsf{B}_i$
2. Set $Tip$ to $\mathsf{B}_\eta$
3. Revert VM state to $\mathsf{B}_\eta$'s state
4. Update $Provisioners$

**Procedure**

$\textit{Fallback}():$
1. $\texttt{for } i = Tip.Height \dots \mathsf{B}_\eta.Height :$
   1. $Mempool = Mempool \cup \{ \mathsf{B}_i.Transactions \}$
   2. $\textbf{Chain}[i] = NIL$
   3. $Blacklist = Blacklist \cup \mathsf{B}_i$
2. $Tip = \mathsf{B}_\eta$
3. $SystemState = VM.Revert(\mathsf{B}_\eta)$
4. $Provisioners = SystemState.Provisioners$

<p><br></p>

## Synchronization
The *synchronization* process allows a node to catch up with a peer's chain. The process, handled by the [*SyncBlock*][sb] procedure, is triggered when receiving a block at a higher height than the $Tip$.

If the triggering block is a valid $Tip$'s successor, it is accepted as the new tip, and the SA loop ([*SALoop*][sl]) is restarted (to sync up the consensus protocol).

Instead, if the triggering block has height higher than the $Tip$'s successor, a protocol is initiated to request to the peer that sent the block all missing blocks. This protocol aims at making the node catch up with the chain of the chosen peer, referred to as *sync peer* (stored by the $syncPeer$ state variable).
The process consists in requesting the missing blocks to the sync peer and waiting for such blocks to be received. When received, each block, in the proper order, is verified and, if valid, accepted to the local chain.

More specifically, the protocol works as follows:
1. PreSync phase ([*PreSync*][ps]):
   1. The node sends a $\mathsf{GetData}$ message (see [Sync Messages][sm]) of type $BlockFromHeight$ requesting the $Tip$'s successor
2. Sync phase ([*StartSync*][ss]):
   1. If the $Tip$'s successor is received:
      1. The node sends a $\mathsf{GetBlocks}$ message to $\mathcal{S}$ containing its local $Tip$;
      2. The sync peer $\mathcal{S}$ replies with an $\mathsf{Inv}$ message containing the hashes of the blocks from the received $Tip$ and its current tip;
      3. When the $\mathsf{Inv}$ message is received, the node requests unknown blocks with a $\mathsf{GetData}$ message;
      4. The peer responds to the $\mathsf{GetData}$ message by sending all requested blocks, one by one, using $\mathsf{Block}$ messages.

In the sync phase, the node stops the SA loop ([*SALoop*][sl]) for efficiency purposes. Note that this is done after receiving a valid $Tip+1$, which is a sign of trustworthiness from the $syncPeer$.
Nonetheless, the sync peer is only given a limited amount of time (defined by [SyncTimeout][env]) to transmit blocks. If the timeout expires, the synchronization is stopped (see [*HandleSyncTimeout*][hst]) and [*SALoop*][sl] is restarted. The timer to keep track of the timeout is set when starting the protocol, and reset each time a valid $Tip$'s successor is provided.

### Environment
The environment of synchronization procedures includes node-level parameters, conditioning the node's behavior during synchronization, and state variables that help keep track of known blocks and handle the synchronization protocol execution.

**Parameters**

| Name             | Value | Description                                        |
|------------------|-------|----------------------------------------------------|
| $PreSyncTimeout$ | $10$  | Pre-Synchronization procedure timeout (in seconds) |
| $SyncTimeout$    | $5$   | Synchronization procedure timeout (in seconds)     |
| $MaxSyncBlocks$  | $50$  | Maximum number of blocks in a sync session         |

**State**

| Name          | Type                 | Description                                                                             |
|---------------|----------------------|-----------------------------------------------------------------------------------------|
| $BlockPool$   | $\mathsf{Block}$ [ ] | List of received blocks with height more than $Tip.Height +1$.                          |
| $Syncing$     | Boolean              | $true$ if we are running a synchronization procedure with some peer, $false$ otherwise. |
| $syncPeer$    | Peer ID              | It contains the ID of the peer the node is synchronizing with.                          |
| $\tau_{Sync}$ | Timestamp            | It contains the time from when the $syncTimeout$ is checked against                     |
| $syncFrom$    | Integer              | The height of the starting block during the synchronization process.                    |
| $syncTo$      | Integer              | The heights of last blocks of synchronization process.                                  |

### Procedures

#### *SyncBlock*
This procedure handles potential successors of the local $Tip$. The procedure accepts valid $Tip$'s successors and is responsible for initiating ([*StartSync*][ss]) and handling synchronization protocols run with nodes peers. Only one synchronization protocol can be run at a time, and only with a single peer (the *sync peer*).

When receiving blocks with higher height than the $Tip$'s successor, they are stored in the $BlockPool$. If receiving a valid $Tip$'s successor, this is accepted along with all valid successors in the $BlockPool$.

If an invalid $Tip$'s successor is received by a sync peer while running the protocol, the protocol is ended.

**Parameters**
- $\mathsf{M^B} = (\mathsf{B}, \mathcal{S})$: received `Block` message

**Algorithm**
<!-- B > Tip+1 -->
1. If $\mathsf{B}$ is higher than the $Tip$'s successor:
   1. If $BlockPool$ has less than $MaxSyncBlocks$
      1. Add $\mathsf{B}$ to the $BlockPool$
   2. If not synchronizing with any peer ($Syncing = false$):
      1. Start pre-synchronization ([*PreSync*][ps]) with peer $\mathcal{S}$
<!-- B = Tip+1 -->
1. Otherwise, if $\mathsf{B}$'s height is $Tip.Height + 1$:
   1. Verify $\mathsf{B}$ ([*VerifyBlock*][vb])
   2. If $\mathsf{B}$ is valid:
      1. Accept $\mathsf{B}$ to the chain
        <!-- Not syncing -->
      2. If not synchronizing with any peer ($Syncing = false$):
          1. If we pre-synced with $\mathcal{S}$ ($syncPeer = \mathcal{S}$)
             1. Start synchronization process ([*StartSync*][ss])
          2. Otherwise:
             1. Propagate $\mathsf{M^B}$
             2. Restart [*SALoop*][sl]
      <!-- Syncing -->
      3. Otherwise (If synchronizing with a peer ($Syncing = true$)) :
          1. Reset sync timeout $\tau_{Sync}$
          2. Accept all consecutive $\mathsf{B}$'s successors in $BlockPool$ ([*AcceptPoolBlocks*][apb])
          3. If the new $Tip$ is $syncTo$
             1. Stop syncing ($Syncing = false$)
             2. Restart [*SALoop*][sl]
   3. Otherwise (if $\mathsf{B}$ is not valid):
     1. If $\mathcal{S}$ is $syncPeer$
         1. Stop syncing ($Syncing = false$)

**Procedure**

$\textit{SyncBlock}(\mathsf{M^B}):$
- $\texttt{set}:$
  - $\mathsf{B},\mathcal{S} \leftarrow \mathsf{M^B}$
<!-- B > Tip+1 -->
1. $\texttt{if } (\mathsf{B}.Height > Tip.Height + 1) :$
   1. $\texttt{if } (|BlockPool| < MaxSyncBlocks):$
      1. $BlockPool = BlockPool \cup \mathsf{B}$
   2. $\texttt{if } (Syncing = false):$
      1. [*PreSync*][ps]$(\mathsf{B}, \mathcal{S})$
<!-- B = Tip+1 -->
1. $\texttt{else if } (\mathsf{B}.Height = Tip.Height+1) :$
   1. $isValid$ = [*VerifyBlock*][vb]$(\mathsf{B}, Tip)$
   2. $\texttt{if } (isValid = true):$
      1. $\texttt{stop}$([*SALoop*][sl])
      2. [*AcceptBlock*][ab]$(\mathsf{B})$
         <!-- Not syncing -->
      3. $\texttt{if } (Syncing = false) :$
          1. $\texttt{if } (\mathcal{S} = syncPeer):$
             1. [*StartSync*][ss]$()$
          2. $\texttt{else}:$
             1. [*Propagate*][mx]$(\mathsf{M^B})$
             2. $\texttt{start}$([*SALoop*][sl])
         <!-- Syncing -->
      4. $\texttt{else}:$
          1. $\tau_{Sync} = \tau_{Now}$
          2. [*AcceptPoolBlocks*][apb]$()$
          3. $\texttt{if } (Tip.Height = syncTo):$
            1. $Syncing = false$
            2. $\texttt{start}$([*SALoop*][sl])
   3. $\texttt{else}:$
       1. $\texttt{if } (\mathcal{S} = syncPeer):$
           1. $Syncing = false$


#### *PreSync*
This procedure requests to a peer the successor of $Tip$ to ensure it knows it before starting the synchronization process. This is done to prevent a peer from stopping our consensus loop just by sending a block in the future. When $Tip+1$ is received from $\mathcal{S}$, the synchronization will start ([*StartSync*][ss]).

A timeout $\tau_{Sync}$ is setup to prevent a peer from blocking our node with sync. $syncPeer$ has maximum $PreSyncTimeout$ to send the first block, and $SyncTimeout$ time to send the following ones. With each valid block the timeout is reset (that is, the peer has $SyncTimeout$ time to send each block).

**Parameters**

- $\mathsf{B}$: the block received
- $\mathcal{S}$: sender peer
 
**Algorithm**

1. Set $syncFrom$ to $Tip$
2. Set $syncTo$ to $\mathsf{B}$'s height or $Tip + MaxSyncBlocks$
3. Set $syncPeer$ to $\mathcal{S}$
4. Send $\mathsf{GetData}$ requesting $Tip$'s successor
5. Start the sync timeout $\tau_{Sync}$
6. Start the sync timeout handler ([*HandleSyncTimeout*][hst])

**Procedure**

$\textit{PreSync}(\mathsf{B}, \mathcal{S}):$
1. $syncFrom = Tip.Height$
2. $syncTo = min(\mathsf{B}.Height, Tip.Height+MaxSyncBlocks)$
3. $syncPeer = \mathcal{S}$
4. [*Send*][mx]$(\mathcal{S}, \mathsf{GetData}(BlockFromHeight, Tip.Height+1))$
5. $\tau_{Sync} = \tau_{Now}$
6. $\texttt{start}($[*HandleSyncTimeout*][hst]$)$

#### *StartSync*
This procedure initiates the synchronization protocol with $syncPeer$ on the base of the block $syncTo$ previously received from this peer.
While syncing, the [*SALoop*][sl] is kept stopped to improve performance.


**Algorithm**
1. Send a $\mathsf{GetBlocks}$ message to $syncPeer$ to request missing blocks
2. Set as "syncing" ($Syncing = true$)

**Procedure**

$\textit{StartSync}():$
1. [*Send*][mx]$(syncPeer, \mathsf{GetBlocks}(Tip.Hash))$
2. $Syncing = true$

#### *HandleSyncTimeout*
This procedure checks if the sync timeout $\tau_{Sync}$ expires when synchronizing or pre-synchronizing with a peer.
If the timeout expires while synchronizing, the synchronization is ended and the [*SALoop*][sl] restarted, if needed. If it expires when pre-syncing, the sync information is reset.

Only a single synchronization process is done at a time, so $\tau_{Sync}$ is unique.

**Procedure**
$\textit{HandleSyncTimeout}():$
- $\texttt{loop}:$
1. $\texttt{if }(Syncing = true):$
   1. $\texttt{if }(\tau_{Now} \gt \tau_{Sync} + SyncTimeout):$
      1. $Syncing = false$
      2. $\texttt{if } (\texttt{running}$([*SALoop*][sl]$) = false)$:
        1. $\texttt{start}$([*SALoop*][sl])
     1. $\texttt{break}$
2. $\texttt{else}:$
   1. $\texttt{if }(\tau_{Now} \gt \tau_{Sync} + PreSyncTimeout):$
      1. $syncPeer = NIL$
      2. $syncFrom = NIL$
      3. $syncTo = NIL$
     1. $\texttt{break}$


#### *AcceptPoolBlocks*
This procedure accepts all successive blocks in $BlockPool$ from height $Tip.Height+1$ until no successor is available. It is called after receiving a valid block at height $Tip.Height+1$ (which updates the $Tip$) to accept previously-collected blocks.

**Algorithm**
1. Get all successive blocks in $BlockPool$ starting from $Tip$'s successor
2. For each retrieved block $\mathsf{B}_i$:
   1. Verify $\mathsf{B}_i$ ([*VerifyBlock*][vb])
      1. If verification fails, stop
   2. Accept $\mathsf{B}_i$ to the chain
   3. Reset sync timeout $\tau_{Sync}$

**Procedure**

$\textit{AcceptPoolBlocks}():$
1. $\boldsymbol{Successors} =$ *getFrom*$(BlockPool, \mathsf{B}_{Tip.Height+1})$
2. $\texttt{for } i = 0 \dots |\boldsymbol{Successors}|{-}1 :$
   1. $isValid$ = [*VerifyBlock*][vb]$(\mathsf{B}_i, Tip)$
      1. $\texttt{if } (isValid = false): \texttt{break}$
   2. [*AcceptBlock*][ab]$(\mathsf{B}_i)$
   3. $\tau_{Sync} = \tau_{Now}$

<!----------------------- FOOTNOTES ----------------------->

[^1]: The current implementation does not fully support all non-Valid votes yet. Most likely, the vote type will be included in the `Attestation` structure.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md -->
[env]: #environment
[ab]:  #acceptblock
[apb]: #acceptpoolblocks

[fin]: #finality
[cs]:  #consensus-state
[lfb]: #last-final-block
[rf]:  #rolling-finality
[crf]: #checkrollingfinality

[fal]: #fallback
[hst]: #handlesynctimeout
[hb]:  #HandleBlock
[syn]: #synchronization
[sb]:  #syncblock
[ps]:  #presync
[ss]:  #startsync
[vb]:  #verifyblock
[vbh]: #verifyblockheader
[va]:  #verifyattestation
[vv]:  #verifyvotes
[hq]:  #HandleQuorum
[mw]:  #makewinning
[vq]:  #VerifyQuorum


<!-- Basics -->
[blk]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#blocks
[lc]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#chain

[sv]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#stepvotes
[ec]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#ExtractCommittee
[gq]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetQuorum
[gsn]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetStepNum
[atts]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestations
[sc]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#subcommittee
[cc]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#countcredits

[inc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#incentives-and-penalties

<!-- Protocol -->
[sa]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#overview
[eb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#emergencyblock
[ieb]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#isemergencyblock
[cenv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#environment
[sl]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#saloop
[sai]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#saiteration

[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md

[ds]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md
[dsp]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md#deterministic-sortition-ds 

<!-- Messages -->
[mx]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#procedures
[bmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#block
[gcmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#getcandidate
[ms]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#signatures
[sm]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#sync-messages

<!-- TODO -->
[xt]:    https://github.com/dusk-network/dusk-protocol/tree/main/virtual-machine
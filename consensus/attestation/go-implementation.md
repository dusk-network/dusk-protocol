# Attestation Phase (Go Implementation)
[`consensus/selection/step.go`]

### Workflow
Input: [SA Parameters](../go-implementation.md#parameters)
Loop :
- **(1) If extracted, generate candidate block and message**
  if *sortition.extracted* (`round`, `step`) : $\hspace{30pt}$ // see `Sortition`
  - **`cMessage`** = ***`GenerateCandidateMessage`*** ( )
    - **`iteration`** = `step/3 + 1`
      **`cbSeed`** = `node`.*sign*(`prevSeed`)
    - **`cBlock`** = ***`GenerateCandidateBlock`*** (`cbSeed`, `round`, `prevHash`, `iteration`, `prevTimestamp`) :
      - /* Fetch transactions from Mempool */
        - if `ExtractionDelaySecs > ConsensusTimeOut` : `delay` = 0
          else : `delay` = `ExtractionDelaySecs`
          `timeout` = `time.now() + delay`
        - while `len(txs) == 0` && `time.now() < timeout` :
          **`txs`** = ***`FetchMempoolTxs`*** (`MaxTxSetSize`) :
          ```
            slippageGasLimit = gasLimit + gasLimit/10
            totalGas = 0

            for tx in mempool.txs   // txs are ordered by gasprice
              estimatedGas = EstimateGasSpent(tx):
                              match tx.type:
                                transfer -> 300_000_000
                                ICC -> 1_200_000_000
              totalGas += estimatedGas
              if totalGas > slippageGasLimit
                continue
              totalSize += tx.size
              if totalSize <= MaxTxSetSize
                txs.append(tx)
              if totalGas >= gasLimit || totalSize >= maxTxsSize
                break
          ```
    
      - /* Execute transactions */ 
        // `txs` is replaced with the set of transactions that are actually executed
      txs, **`stateHash`** = `node`.*execute*(txs, round, blockGasLimit)

      - /* Create Candidate Block */
        - **`timestamp`** = `time.Now().Unix()`
          ```go
            maxTimestamp = prevTimestamp + MaxBlockTime
            if round > 1 && prevTimestamp > 0 :
              if timestamp < prevTimestamp :
                timestamp = prevTimestamp
              else if timestamp > maxTimestamp :
                timestamp = maxTimestamp
          ```
        - **`cbHeader`** = 
          ```
            BlockHeader {
                Version:            0,
                Timestamp:          timestamp,
                Height:             round,
                PrevBlockHash:      prevBlockHash,
                GeneratorBlsPubkey: node.BLSPubKey,
                TxRoot:             nil,
                Seed:               cbSeed,
                Certificate:        nil,
                StateHash:          stateHash,
                GasLimit:           blockGasLimit,
                Iteration:          iteration,
            }
          ```
        - **`cBlock`** = 
          ```
            Block {
                Header: cbHeader,
                Txs:    txs,
            }
          ```
        - `cBlock.Header.`**`TxRoot`** = *CalculateTxRoot*(`txs`)  // Builds the tx merkle tree and returns the root
        - `cBlock.Header.`**`Hash`** = 
          ```
            Hash(
                cBlock.Header.Version
                cBlock.Header.Height
                cBlock.Header.Timestamp
                cBlock.Header.PrevBlockHash
                cBlock.Header.Seed
                cBlock.Header.StateHash
                cBlock.Header.GeneratorBlsPubkey
                cBlock.Header.TxRoot
                cBlock.Header.GasLimit
                cBlock.Header.Iteration
            )
          ```
      - return `cBlock` 

  - /* Create NewBlock Message */
    - **`nbHeader`** = 
      ``` 
        Header {
            PubKeyBLS:     node.BLSPubKey,
            Round:         round,
            Step:          step,
            BlockHash:     cBlock.Header.Hash
        }
      ```
    - **`nbMsg`** = 
      ```
        NewBlock {
            hdr:        nbHeader,
            PrevHash:   prevHash,
            Candidate:  cBlock,
            SignedHash: node.Sign(nbHeader)
        }
      ```

  - /* Broadcast NewBlock message */
    ***broadcast***(`nbMsg`)               // see `Kadcast`

  - /* Emit newBlock event */
    ***emitEvent***(newBlock(`nbMsg`))

- /* Handle Attestation events */
  match `queue`.*GetEvents*(`round`, `step`):
  - /* **Handle NewBlock message** */
    case NewBlock (`nbMsg`) :
    - // Check message is for current round and step
    `shouldProcess` = *checkMessage*(`nbMsg`)
      <!-- ```
        if nbMsg.round != round 
          || nbMsg.step != step
          || nbMsg.topic != topics.NewBlock :
            shouldProcess = false
        else:
            shouldProcess = true
      ``` -->
    - // Process new block
      if `shouldProcess` :
      - **`newBlock`** = *collectNewBlock*(`nbMsg`) 
      - // Verify block and message validity
        *verifyBlock*(`newBlock`) :
          - // Check message signer is the extracted block's producer
          Assert ( *sortition.extracted*(`newBlock.PubKeyBLS`) )
          - // Check signature validity
          Assert ( *VerifyBLSSignature*(`nbMsg.PubKeyBLS`,`nbMsg.signature`) )
          - // Check block hash is correct
          Assert ( *Hash*(`newBlock`) == `Block.Header.BlockHash` )
          - // Check transaction root is correct
          Assert ( *MerkleTree*(`newBlock.txs`).`root` == `newBlock.Header.TxRoot`)
          - // Check block's hash and message BlockHash match
          Assert ( *Hash*(`newBlock`) == `nbMsg.BlockHash` )
      - // Store message in the DB
        StoreMessage(`nbMsg`)

    - // End Attestation phase and start Reduction with Block
      *InitFirstReduction*(`newBlock`)

  - /* **Handle Timeout** */
    case timeoutExpired
    - // End Attestation phase and start Reduction with an empty block
      *InitFirstReduction*(*EmptyBlock*())
  
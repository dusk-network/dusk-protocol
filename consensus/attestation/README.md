# Attestation Phase

In [Succinct Attestation](../README.md), the *Attestation* phase is the first step in an [*iteration*](../README.md#workflow).

In this phase, a provisioner is selected to generate a new *candidate block* and broadcast it with a `NewBlock` message. 
Upon receiving such a message, other Provisioners verify its validity and, if valid, switch to the [*1st Reduction*](../reduction) phase to vote on the block.

### Algorithm
This phase is run in a loop by all provisioner nodes, producing a new candidate block when selected as generator, or processing incoming `NewBlock` messages otherwise.

Inputs: [SA Parameters](../README.md#parameters)
Procedure:

- loop:
  - (1) **Check if extracted as block generator** -- see [*Deterministic Sortition*](../sortition/README.md)
    - If extracted:
      - (a) **Generate candidate block `CB`** -- see [*generateCandidateBlock()*](#generatecandidateblock)
      - (b) **Create `NewBlock` message `NBM`** -- see [*createNewBlockMessage()*](#createnewblockmessage)
      - (c) **Broadcast `NBM`** -- see [*Kadcast*]()
      - (d) **Emit `NewBlock_Message` event**

    <!-- Insert vertical spacing -->
    <div><br></div>

    - (2) **Handle events** :
        - **`NewBlock` message `NBM` received** (for current `round`/`step`):
        - (a) **Verify message and block validity** -- see [*verifyNewBlock()*](#verifynewblock)
        - (b) **Store `NBM` in the local DB** <!-- TODO?: Storage -->
        - (c) **Start First Reduction with `NBM.Block`** -- see [First Reduction](../reduction)
        - **Timeout expired**:
        - (a) **Start First Reduction with `EmptyBlock`** -- see [First Reduction](../reduction)


### Subroutines

##### generateCandidateBlock()
Procedure:
 - (1) **Fetch transactions from Mempool** -- see [*Mempool*]() <!-- TODO -->
   - `transactions` = [*selectTransactions()*](#selecttransactions)
 - (2) **Compute new state hash** -- see [*VM and State*]() <!-- TODO -->
   - `newStateHash` = vm.[*execute(`transactions`)*]() <!-- TODO -->
 - (3) **Create block**:
   - (a) Compute block fields:
     - **Set block timestamp** to current time in Unix format
       - `timestamp` = `time.Now().Unix()`
     - **Compute block seed** as the signature over the previous one -- see [*Deterministic Sortition*](../sortition/README.md)
       - `newSeed` = `node.sign(prevSeed)` 
     - **Compute current iteration** -- see [*Succinct Attestation*](../README.md)
       - `iteration` = `step/3 + 1`
     - **Compute transaction Merkle tree root** <!-- TODO: see [MerkleTree]() ? -->
       - `txRoot` = `MerkleTree(transactions).root` 
     - **Compute Header Hash**
       - `HeaderHash` = `Hash(Version||Height||Timestamp||GasLimit||Iteration||PrevBlockHash||GeneratorPubKey||TxRoot||Seed||StateHash)`
   - (b) **Create candidate block** :
        `candidateBlock` =
        ```
          Block {
              Header: {
                  Version:            0,
                  Height:             round,
                  Timestamp:          timestamp,
                  GasLimit:           blockGasLimit,
                  Iteration:          iteration,
                  PreviousBlockHash:  prevBlockHash,
                  GeneratorPublicKey: node.PubKey,
                  TransactionRoot:    txRoot,
                  Seed:               newSeed,
                  StateHash:          newStateHash,
                  HeaderHash:         HeaderHash,
                  
                  Certificate:        nil,
              },
              Txs:    transactions,
          }
        ```

<p><br></p>

##### selectTransactions()
The selection of the transactions to be included in the block is arbitrary and can vary between different implementations.
Typically, the block producer will aim at maximizing its profits by selecting transactions paying higher gas price.
In this respect, it can be said that transactions paying higher gas prices will be prioritized by block producers, and hence will be included in the blockchain earlier.

To ease this process, transactions in the Mempool are ordered by their gas price.
<!-- TODO: -- see our implementation -->

##### createNewBlockMessage()
This procedure creates a `NewBlock` message with the new candidate block and the provisioner's signature.

Inputs: `candidateBlock`
Procedure:
  - (1) **Create message header**
    - `messageHeader` = 
      ``` 
        Header {
            PubKeyBLS:     node.BLSPubKey,
            Round:         round,
            Step:          step,
            BlockHash:     candidateBlock.Header.Hash
        }
      ```
  - (2) **Sign message header**
    - `signedHash` = `node.Sign(messageHeader)`
  - (3) **Create NewBlock Message**
    - `NewBlockMessage` = 
      ```
        NewBlock {
            hdr:        nbHeader,
            PrevHash:   prevHash,
            Candidate:  candidateBlock,
            SignedHash: signedHash
        }
      ```
  - (4) Output `NewBlockMessage`

##### verifyNewBlock()
Verifies the validity of a NewBlock message.

Inputs: `NewBlockMessage`
Procedure:
 - (1) **Verify signer is expected block producer**
 - (2) **Check signature validity**
 - (3) **Check block hash**
 - (4) **Check transaction root**
 - (5) **Check message and block hash match**


<!-- TODO:
Write everything in <pre> ?
Ex:
    <pre>
    Procedure:
    - (1) <b>Fetch transactions from Mempool</b> -- see <a href="">Mempool</a>
    </pre>
 -->
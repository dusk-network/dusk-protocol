## Block Structure
```
type Block struct {
    Header *Header                     `json:"header"`
    Txs    []transactions.ContractCall `json:"transactions"`
}
```

## Header Structure
```
type Header struct {
    Version   uint8  `json:"version"`   // Block version byte
    Height    uint64 `json:"height"`    // Block height
    Timestamp int64  `json:"timestamp"` // Block timestamp
    GasLimit  uint64 `json:"gaslimit"`  // Block gas limit
    Iteration uint8  `json:"iteration"` // Iteration at which block is produced

    PrevBlockHash      []byte `json:"prev-hash"`  // Hash of previous block (32 bytes)
    Seed               []byte `json:"seed"`       // Marshaled BLS signature or hash of the previous block seed (32 bytes)
    GeneratorBlsPubkey []byte `json:"generator"`  // Generator BLS Public Key (96 bytes)
    TxRoot             []byte `json:"tx-root"`    // Root hash of the merkle tree containing all txes (32 bytes)
    StateHash          []byte `json:"state-hash"` // Root hash of the Rusk Contract Storage state

    Hash []byte `json:"hash"` // Hash of all previous fields

    *Certificate `json:"certificate"` // Block certificate
}
```
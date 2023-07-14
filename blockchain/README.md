<!-- TODO: add JSON names? -->

## Block Structure

| Field                 | Type                      | Size      | Description          |
|-----------------------|---------------------------|-----------|----------------------|
| Header                | Block Header              | ?         | Block header         |
| Transactions          | Vector [ Transactions ]   | variable  | Block height         |

## Header Structure

| Field                 | Type   | Size      | Description                               |
|-----------------------|--------|-----------|-------------------------------------------|
| Version               | uint   | 8 bits    | Block version                             |
| Height                | uint   | 64 bits   | Block height                              |
| Timestamp             | int    | 64 bits   | Block timestamp                           |
| GasLimit             | uint   | 64 bits   | Block gas limit                           |
| Iteration             | int    | 8 bits    | Iteration at which the block was produced |
| Previous Block Hash   | string | 32 bytes  | Hash of previous block                    |
| Seed                  | string | 48 bytes  | Signature of the previous block's seed    |
| Generator Public Key  | string | 96 bytes  | Generator Public Key                      |
| Transaction Root      | string | 32 bytes  | Root of transactions Merkle tree          |
| State Hash            | string | 32 bytes ?| Root of contracts state Merkle tree       |
| Header Hash           | string | 32 bytes ?| Hash of all previous fields               |
| Certificate           |    ?   |     ?     |    ?                                      |

## Transaction Structure

| Field   | Type   | Size      | Description         |
|---------|--------|-----------|---------------------|
| Version | uint   | 32 bits   | Transaction version |
| Type    | uint   | 32 bits   | Transaction type    |
| Payload | byte[] | variable  | Transaction payload |
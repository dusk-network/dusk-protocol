<!-- TODO: add JSON names? -->

## Block Structure

| Field          | Type                      | Size      | Description          |
|----------------|---------------------------|-----------|----------------------|
| $Header$       | Block Header              | ?         | Block header         |
| $Transactions$ | Vector [ Transactions ]   | variable  | Block height         |

## Header Structure

| Field                 | Type   | Size      | Description                               |
|-----------------------|--------|-----------|-------------------------------------------|
| $Version$               | uint   | 8 bits    | Block version                             |
| $Height$                | uint   | 64 bits   | Block height                              |
| $Timestamp$             | int    | 64 bits   | Block timestamp                           |
| $GasLimit$             | uint   | 64 bits   | Block gas limit                           |
| $Iteration$             | int    | 8 bits    | Iteration at which the block was produced |
| $PreviousBlock$   | string | 32 bytes  | Hash of previous block                    |
| $Seed$                  | string | 48 bytes  | Signature of the previous block's seed    |
| $Generator$  | string | 96 bytes  | Generator Public Key                      |
| $TransactionRoot$      | string | 32 bytes  | Root of transactions Merkle tree          |
| $StateRoot$            | string | 32 bytes ?| Root of contracts state Merkle tree       |
| $Hash$               | Sha3-256 Hash | 32 bytes | Hash of the header (ex. this and $Certificate$)               |
| $Certificate$           |    ?   |     ?     |    ?                                      |

For the sake of readability, we refer to header fields omitting the $.Header$ as this does not generate any ambiguity. For instance $\mathcal{B}.Seed$ will correspond to $\mathcal{B}.Header.Seed$
<!-- TODO: if we use $\mathcal{H}^B$ we can avoid this -->

### Notation
We denote the header of a block $\mathcal{B}$ as $\mathcal{H}_\mathcal{B}$.

We define a *block's hash* as:

$$\eta_\mathcal{B} = H_{SHA3}(Version||Height||Timestamp||GasLimit||Iteration||\\PreviousBlock||Seed||Generator||TransactionRoot||StateRoot)$$


## Transaction Structure

| Field     | Type   | Size      | Description         |
|-----------|--------|-----------|---------------------|
| $Version$ | uint   | 32 bits   | Transaction version |
| $Type$    | uint   | 32 bits   | Transaction type    |
| $Payload$ | byte[] | variable  | Transaction payload |
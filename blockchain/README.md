<!-- TODO: add JSON names? -->

## Block Structure

| Field          | Type            | Size     | Description        |
|----------------|-----------------|----------|--------------------|
| $Header$       | Block Header    | ?        | Block header       |
| $Transactions$ | Transaction [ ] | variable | Block transactions |
<!-- | $Final$        | Boolean         | 1 bit    | Final state ($true$ if final, $false$ otherwise) | -->

## BlockHeader Structure
| Field             | Type               | Size      | Description                               |
|-------------------|--------------------|-----------|-------------------------------------------|
| $Version$         | Unsigned Integer   | 8 bits    | Block version                             |
| $Height$          | Unsigned Integer   | 64 bits   | Block height                              |
| $Timestamp$       | Integer            | 64 bits   | Block timestamp                           |
| $GasLimit$        | Unsigned Integer   | 64 bits   | Block gas limit                           |
| $Iteration$       | Integer            | 8 bits    | Iteration at which the block was produced |
| $PreviousBlock$   | Sha3 Hash          | 32 bytes  | Hash of previous block                    |
| $Seed$            | Signature          | 48 bytes  | Signature of the previous block's seed    |
| $Generator$       | Public Key         | 96 bytes  | Generator Public Key                      |
| $TransactionRoot$ | Blake3 Hash        | 32 bytes  | Root of transactions Merkle tree          |
| $StateRoot$       | Sha3 Hash          | 32 bytes  | Root of contracts state Merkle tree       |
| $Hash$            | Sha3 Hash          | 32 bytes  | Hash of previous fields                   |
| $Certificate$     | [StepVotes][sv] [] | 112 bytes | Reduction votes                           |

For the sake of readability, we refer to header fields omitting the $.Header$ as this does not generate any ambiguity. For instance $\mathsf{B}.Seed$ will correspond to $\mathsf{B}.Header.Seed$
<!-- TODO: if we use $\mathsf{H}^B$ we can avoid this -->

### Notation
We denote the header of a block $\mathsf{B}$ as $\mathsf{H_B}$.

We define a *block's hash* as:

$$\eta_\mathsf{B} = Hash_{SHA3}(Version||Height||Timestamp||GasLimit||Iteration||\\PreviousBlock||Seed||Generator||TransactionRoot||StateRoot)$$


## Transaction Structure

| Field     | Type   | Size      | Description         |
|-----------|--------|-----------|---------------------|
| $Version$ | uint   | 32 bits   | Transaction version |
| $Type$    | uint   | 32 bits   | Transaction type    |
| $Payload$ | byte[] | variable  | Transaction payload |

<!--  -->
[sv]: ../consensus/reduction/README.md#stepvotes
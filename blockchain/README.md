# Blockchain
This section describes the formal definition of the block and transaction structures

#### ToC
- [`Block` Structure](#block-structure)
- [`BlockHeader` Structure](#blockheader-structure)
  - [Block Notation](#block-notation)
- [`Transaction` Structure](#transaction-structure)


## Block Structure

| Field          | Type            | Size      | Description        |
|----------------|-----------------|-----------|--------------------|
| $Header$       | `BlockHeader`    | 112 bytes | Block header       |
| $Transactions$ | `Transaction` [ ] | variable  | Block transactions |
<!-- TODO? | $Final$        | Boolean         | 1 bit    | Final state ($true$ if final, $false$ otherwise) | -->

## BlockHeader Structure
| Field                  | Type                   | Size       | Description                               |
|------------------------|------------------------|------------|-------------------------------------------|
| $Version$              | Unsigned Integer       | 8 bits     | Block version                             |
| $Height$               | Unsigned Integer       | 64 bits    | Block height                              |
| $Timestamp$            | Integer                | 64 bits    | Block timestamp                           |
| $GasLimit$             | Unsigned Integer       | 64 bits    | Block gas limit                           |
| $Iteration$            | Integer                | 8 bits     | Iteration at which the block was produced |
| $PreviousBlock$        | Sha3 Hash              | 32 bytes   | Hash of previous block                    |
| $Seed$                 | Signature              | 48 bytes   | Signature of the previous block's seed    |
| $Generator$            | Public Key             | 96 bytes   | Generator Public Key                      |
| $TransactionRoot$      | Blake3 Hash            | 32 bytes   | Root of transactions Merkle tree          |
| $StateRoot$            | Sha3 Hash              | 32 bytes   | Root of contracts state Merkle tree       |
| $PrevBlockCertificate$ | [Certificate][cert]    | 112 bytes  | Reduction votes of previous block         |
| $Hash$                 | Sha3 Hash              | 32 bytes   | Hash of previous fields                   |
| $Certificate$          | [Certificate][cert]    | 112 bytes  | Reduction votes                           |
| $FailedIterations$     | [Certificate][cert][ ] | 0-28448 bytes (27.75 KB) | Aggregated votes of failed iterations |

The $BlockHeader$ structure has a variable total size of 522 to 28970 bytes (28.8 KB).
This is reduced to 410-28448 bytes for a [*candidate block*][cb], since $Certificate$ is missing.

### Block Notation
We denote the header of a block $\mathsf{B}$ as $\mathsf{H_B}$.

We define a *block's hash* as:

<!-- TODO: define \eta as function: \eta(B) -->
$$\eta_\mathsf{B} = Hash_{SHA3}(Version||Height||Timestamp||GasLimit||Iteration||PreviousBlock||Seed||Generator||TransactionRoot||StateRoot||PrevBlockCertificate)$$

<!-- TODO: define block's round and iteration: r_{\mathsf{B}^p},s_{\mathsf{B}^p} -->

## Transaction Structure

| Field     | Type    | Size      | Description         |
|-----------|---------|-----------|---------------------|
| $Version$ | uint    | 32 bits   | Transaction version |
| $Type$    | uint    | 32 bits   | Transaction type    |
| $Payload$ | byte[ ] | variable  | Transaction payload |

<!------------------------- LINKS ------------------------->
<!-- Consensus -->
[cb]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#candidate-block
<!-- Reduction -->
[sv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md#stepvotes
<!-- Ratification -->
[cert]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification#certificate
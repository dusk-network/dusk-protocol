# Blockchain
This section describes the formal definition of the Dusk chain, block, and transaction structures.

#### ToC
- [Chain](#chain)
- [`Block`](#block)
- [`BlockHeader`](#blockheader)
- [`Transaction`](#transaction)

## Chain
The local copy of the blockchain stored by a node is defined as a vector of [blocks](#block) along with their *consensus status* label (see [Block Finality][fin]):

$$\textbf{Chain}: [(\mathsf{B}_0, Status_0), \text{ }\dots, (\mathsf{B}_n, Status_n)],$$

where $\mathsf{B}_0 = Genesis$ and $Status_0 = \text{"Final"}$ (that is, the genesis block is final by definition).

**Notation**
We use $\textbf{Chain}[i]$ to indicate the $i$th element of the chain.

<!-- TODO: define Genesis and Tip here; review use of Tip in procedures (we can use Chain instead) -->

## Block

| Field            | Type             | Size      | Description        |
|------------------|------------------|-----------|--------------------|
| $Header$         | [`BlockHeader`][bh]    | 112 bytes | Block header       |
| $Transactions$   | [`Transaction`][tx][ ] | variable  | Block transactions |

## BlockHeader

| Field                  | Type                     | Size       | Description                               |
|------------------------|--------------------------|------------|-------------------------------------------|
| $Version$              | Unsigned Integer         | 8 bits     | Block version                             |
| $Height$               | Unsigned Integer         | 64 bits    | Block height                              |
| $Timestamp$            | Integer                  | 64 bits    | Block timestamp                           |
| $GasLimit$             | Unsigned Integer         | 64 bits    | Block gas limit                           |
| $Iteration$            | Integer                  | 8 bits     | Iteration at which the block was produced |
| $PreviousBlock$        | Sha3 Hash                | 32 bytes   | Hash of previous block                    |
| $Seed$                 | Signature                | 48 bytes   | Signature of the previous block's seed    |
| $Generator$            | Public Key               | 96 bytes   | Generator Public Key                      |
| $TransactionRoot$      | Blake3 Hash              | 32 bytes   | Root of transactions Merkle tree          |
| $StateRoot$            | Sha3 Hash                | 32 bytes   | Root of contracts state Merkle tree       |
| $PrevBlockCertificate$ | [`Certificate`][cert]    | 112 bytes  | Reduction votes of previous block         |
| $Hash$                 | Sha3 Hash                | 32 bytes   | Hash of previous fields                   |
| $Certificate$          | [`Certificate`][cert]    | 112 bytes  | Reduction votes                           |
| $FailedIterations$     | [`Certificate`][cert][ ] | 0-28448 bytes (27.75 KB) | Aggregated votes of failed iterations |

The $BlockHeader$ structure has a variable total size of 522 to 28970 bytes (28.8 KB).
This is reduced to 410-28448 bytes for a [*candidate block*][cb], since $Certificate$ is missing.

**Notation**
We denote the header of a block $\mathsf{B}$ as $\mathsf{H_B}$.

We define a *block's hash* as:

<!-- TODO: define \eta as function: \eta(B) -->
$$\eta_\mathsf{B} = Hash_{SHA3}(Version||Height||Timestamp||GasLimit||Iteration||PreviousBlock||Seed||Generator||TransactionRoot||StateRoot||PrevBlockCertificate)$$

<!-- TODO: define block's round and iteration: r_{\mathsf{B}^p},s_{\mathsf{B}^p} -->

## Transaction

| Field     | Type    | Size      | Description         |
|-----------|---------|-----------|---------------------|
| $Version$ | Unsigned Integer    | 32 bits   | Transaction version |
| $Type$    | Unsigned Integer    | 32 bits   | Transaction type    |
| $Payload$ | Byte[ ] | variable  | Transaction payload |

<!------------------------- LINKS ------------------------->
[b]:  #block
[bh]: #blockheader
[c]:  #chain
[tx]: #transaction

<!-- Consensus -->
[cb]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#candidate-block
[cert]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#certificates
[sv]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#stepvotes
<!-- Chain Management -->
[fin]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#finality
[cs]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#consensus-state

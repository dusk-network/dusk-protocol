# Protocol Messages
This section describes the network messages exchanged by nodes to participate in the SA consensus protocol and exchange blocks and transactions.

**ToC**
  - [Consensus](#consensus)
      - [Sender](#sender)
    - [Preamble](#preamble)
      - [`ConsensusInfo`](#consensusinfo)
    - [Signatures](#signatures)
    - [Procedures](#procedures)
      - [`SignInfo`](#signinfo)
    - [Messages](#messages)
      - [`Candidate`](#candidate)
      - [`Validation`](#validation)
      - [`Ratification`](#ratification)
      - [`Quorum`](#quorum)
    - [Procedures](#procedures-1)
      - [*CMsg*](#cmsg)
  - [Data Exchange](#data-exchange)
    - [Structures](#structures)
      - [`Inv`](#inv)
      - [`InvItem`](#invitem)
      - [`InvParam` Enum](#invparam-enum)
    - [Messages](#messages-1)
      - [`Block`](#block)
      - [`Transaction`](#transaction)
      - [`GetMempool`](#getmempool)
      - [`GetBlocks`](#getblocks)
      - [`GetResource`](#getresource)
        - [Resource Discovery](#resource-discovery)
        - [Direct Request](#direct-request)
      - [`Inv`](#inv-1)
  - [Procedures](#procedures-2)
      - [*Broadcast*](#broadcast)
      - [*Receive*](#receive)
      - [*Propagate*](#propagate)
      - [*Send*](#send)


## Consensus
To run the SA protocol, nodes exchange four main types of messages:
- $\mathsf{Candidate}$: it stores a candidate block for a specific round and iteration; it is used during the [Proposal][prop] step;
- $\mathsf{Validation}$: it contains the vote of a member of the voting committee for a [Validation][val] step;
- $\mathsf{Ratification}$: it contains the vote of a member of the voting committee for a [Ratification][rat] step;
- $\mathsf{Quorum}$: it contains the aggregated votes of a specific iteration's Validation and Ratification steps; it is generated at the end of an SA iteration if a quorum was reached in the Ratification step;

Consensus messages are composed of a [`ConsensusInfo`][cinf] structure, some fields specific to the message type, and, with the exception of the `Quorum` message, a [`SignInfo`][sinf] structure, containing the public key and signature of the provisioner that created the message. 

#### Sender
All messages include a $Sender$ field indicating the network identity (e.g. the IP address) of the peer from which the message was received. 
For the sake of readability, this field is omitted from the structure definition.

### Preamble
All consensus messages includes the essential information to identify the round, iteration, and branch they belong to.
We include all this information in the `ConsensusInfo` structure, which is the first field of all consensus messages.

#### `ConsensusInfo`
This structure is used for all consensus messages to identify the round and iteration they refer to. 
The information included is the previous block hash, and the round and iteration numbers. 

That the presence of the previous block allows to distinguish between messages for the same round and iteration but from different forks[^1].

The structure is defined as follows:

| Field       | Type         | Size     | Description         |
|-------------|--------------|----------|---------------------|
| $PrevHash$  | [SHA3][hash] | 32 bytes | Previous block hash |
| $Round$     | Unsigned Int | 64 bits  | Round number        |
| $Iteration$ | Unsigned Int | 64 bits  | Iteration number    |

The structure's total size is 48 bytes.


### Signatures
To ensure integrity and authenticity of votes, each message is digitally signed by the sender.

In particular, all messages include a signature of the following information:
- the [`ConsensusInfo`][cinf] structure
- the message-specific payload (e.g., the candidate hash for `Candidate` messages, or the `Vote` for Validation and Ratification messages)

This *message signature* is included in the $Signature$ field.

Formally, given a message $\mathsf{M}$, we define its signature $\eta_\mathsf{M}$ as

$$\eta_\mathsf{M} = Sign_{BLS}([Blake2B][hash](\upsilon_\mathsf{M}), sk),$$

where $sk$ is the secret key paired with the message's $Signer$, and $\upsilon$ is the *signature value*, that is, the content of the message being signed[^2].
Note that the signature value depends on the message type.

### Procedures
With respect to message signatures, we also define the following procedures:
 - $SignMessage(\mathsf{M})$: takes a message $\mathsf{M}$ and outputs the message signature $\sigma_\mathsf{M}$;
 - $VerifyMessage(\mathsf{M})$: takes a message $\mathsf{M}$ and outputs $true$ if $\sigma_\mathsf{M}$ is a valid signature of $Signer$ over the signature value $\upsilon_\mathsf{M}$.

#### `SignInfo`
This structure contains the public key of the provisioner creating the message and its signature over the message.

| Field       | Type              | Size      | Description                       |
|-------------|-------------------|-----------|-----------------------------------|
| $Signer$    | BLS Key           | 96 bytes  | Public Key of the message creator |
| $Signature$ | BLS Signature     | 48 bytes  | Message signature                 |

The structure's total size is 144 bytes.

The $Signer$ field identifies the provisioner sending the message. For instance, in a `Candidate` message, this field identifies the block generator, while, in a `Validation` message, it identifies the committee member who cast the vote.

The $Signature$ field is not just the signature of the whole message but varies depending on the message type. This signature is used for the consensus protocol, especially for `Validation` and `Ratification` messages, whose signature is used to prove the vote and, if a quorum is reached, to certify the quorum in an iteration.


### Messages

#### `Candidate`
This message is used by a block generator to broadcast a candidate block.

The message has the following structure:

| Field           | Type                    | Size           | Description     |
|-----------------|-------------------------|----------------|-----------------|
| $ConsensusInfo$ | [`ConsensusInfo`][cinf] | 48 bytes       | Consensus info  |
| $Candidate$     | [`Block`][b]            | 410-1306 bytes | Candidate block |
| $SignInfo$      | [`SignInfo`][sinf]      | 144 bytes      | Signature info  |

The message has a variable size of 192 bytes plus the block size.

The message's [signature value][sigs] $\upsilon_\mathcal{M^C}$ is:

$$\upsilon_\mathcal{M^C} = (ConsensusInfo || \eta_{Candidate})$$


#### `Validation`
This message is used by a member of a [Validation][val] committee to cast a vote on a candidate block. 

The message has the following structure:

| Field           | Type                    | Size      | Description     |
|-----------------|-------------------------|-----------|-----------------|
| $ConsensusInfo$ | [`ConsensusInfo`][cinf] | 48 bytes  | Consensus info  |
| $Vote$          | [`Vote`][vote]          | 33 byte   | Validation vote |
| $SignInfo$      | [`SignInfo`][sinf]      | 144 bytes | Signature info  |

The message has a total size of 225 bytes.

In this message, $Vote$ can be $Valid$, $Invalid$, or $NoCandidate$.

The message's [signature value][sigs] $\upsilon_\mathcal{M^V}$ is:

$$\upsilon_\mathcal{M^V} = (ConsensusInfo || Vote || ValStep)$$


#### `Ratification`
This message is used by a member of a [Ratification][rat] committee to cast a vote on a candidate block.

The message has the following structure:
| Field             | Type                    | Size      | Description                 |
|-------------------|-------------------------|-----------|-----------------------------|
| $ConsensusInfo$   | [`ConsensusInfo`][cinf] | 48 bytes  | Consensus info              |
| $Vote$            | [`Vote`][vote]          | 33 byte   | Ratification vote           |
| $ValidationVotes$ | [`StepVotes`][sv]       | 56 byte   | Aggregated Validation votes |
| $Timestamp$       | Unsigned Int            | 64 bits   | Timestamp in Unix format    |
| $SignInfo$        | [`SignInfo`][sinf]      | 144 bytes | Signature info              |

The message has a total size of 281 bytes.

The $Timestamp$ field is reserved for a future feature.

The message's [signature value][sigs] $\upsilon_\mathcal{M^R}$ is:

$$\upsilon_\mathcal{M^R} = (ConsensusInfo || Vote || RatStep)$$


#### `Quorum`
This message is used at the end of a successful SA iteration to communicate to the network that a quorum of votes has been reached. The message is generated by any node collecting a quorum of votes for Ratification. The payload includes the `Attestation` with votes from both [Validation][val] and [Ratification][rat] steps.

| Field           | Type                    | Size      | Description           |
|-----------------|-------------------------|-----------|-----------------------|
| $ConsensusInfo$ | [`ConsensusInfo`][cinf] | 48 bytes  | Consensus info        |
| $Vote$          | [`Vote`][vote]          | 33 byte   | Quorum vote           |
| $Attestation$   | [`Attestation`][att]    | 112 bytes | Iteration attestation |

The message has a total size of 249 bytes.

Note that this message is not signed, since the $Attestation$ is already made of aggregated signatures, which can be verified against the $ConsensusInfo$ and the $Vote$. As a consequence, any node of the network can broadcast this message, when collecting a quorum of Ratification votes.

### Procedures

#### *CMsg*
We define a common *CMsg* (Consensus Message) constructor procedure, which is used to create a consensus message:

$Msg(\mathsf{T}, f_1,\dots,f_2):$
1. $\mathsf{ConsensusInfo} = (\eta_{Tip}, R, I)$
2. $\sigma_{M^T} = Sign_{BLS}(\upsilon_\mathsf{M^T}, sk_\mathcal{N})$
3. $\mathsf{SignInfo} = (pk_\mathcal{N}, \sigma_{M^T})$
4. $\mathsf{M^T} = (\mathsf{ConsensusInfo}, f_1, \dots, f_n, SignInfo)$
5. $\texttt{output } \mathsf{M^T}$

The procedure uses the local consensus parameters ($Tip$, $R$, and $I$) and the node secret key $sk_\mathcal{N}$ to generate the message header and then builds the message with the other parameters $(f_1,\dots,f_2)$.

We assume the procedure is able to automatically generate $\upsilon_\mathsf{M^T}$ based on the message type and inputs.

$\mathsf{T}$ indicates the message type: $\mathsf{Candidate}$, $\mathsf{Validation}$, $\mathsf{Ratification}$, or $\mathsf{Quorum}$.

<p><br></p>

## Data Exchange
These messages are used by nodes to request missing data objects to their peers.

We define the following messages:
- $\mathsf{Block}$: it contains a [`Block`][b];
- $\mathsf{Transaction}$: it contains a [`Transaction`][tx]

- $\mathsf{GetMempool}$: it requests all transactions in the node's *Mempool*; it has no payload;
- $\mathsf{GetBlocks}$: it requests all successors of a certain block; it is used to synchronize the chain with a peer;
- $\mathsf{GetResource}$: it requests a resource (transaction or block);
- $\mathsf{Inv}$: it contains a list of available resources (transactions and/or blocks);

### Structures

#### `Inv`
This structure contains a list of inventory items (transactions and blocks).
It is used both to advertise known resources as well as to request them.

| Field        | Type               | Size     | Description             |
|--------------|--------------------|----------|-------------------------|
| $InvList$    | [`InvItem`][ii][ ] | variable | List of inventory items |
| $MaxEntries$ | Unsigned Int       | 2 bytes  | Maximum number of items to send (used for $\mathsf{GetResource}$ messages) |

#### `InvItem`

| Field      | Type         | Size     | Description                    |
|------------|--------------|----------|--------------------------------|
| $InvType$  | Unsigned Int | 1 byte   | The type of the inventory item |
| $InvParam$ | `InvParam`   | 40 bytes | The item ID                    |

$InvType$ can have the following values:
  - $0$: $MempoolTx$
  - $1$: $BlockFromHash$
  - $2$: $BlockFromHeight$
  - $3$: $CandidateFromHash$

$InvParam$ is an Enum and depends on $InvType$

#### `InvParam` Enum

| Variant  | Data         | Size     | Description                                                                   |
|----------|--------------|----------|-------------------------------------------------------------------------------|
| $Hash$   | $SHA3$       | 32 bytes | A hash value, used with $BlockFromHash$, $CandidateFromHash$, and $MempoolTx$ |
| $Height$ | Unsigned Int | 8 bytes  | A block height, used with $BlockFromHeight$                                   |

The enum size is 40 bytes, given by: 1 "discriminant" byte, 32 bytes for the biggest possible value ($Hash$), plus 7 bytes to align the field to 8 bytes.


### Messages

#### `Block`
It contains a [`Block`][b] structure. It is sent in response to a [`GetBlocks`][gbmsg] message or to a [`GetResource`][grmsg] message requesting a block.


#### `Transaction`
It contains a [`Transaction`][tx] structure. It is sent in response to a [`GetResource`][grmsg] message requesting a transaction.


#### `GetMempool`
It requests the list of all transactions in a peer's Mempool. It has no payload.

The response to this message is an $\mathsf{Inv}$ message containing a vector of type $MempoolTx$ containing the list of Transaction IDs ($Hash$).

The expected message flow is the following: $\mathsf{GetMempool}$ -> $\mathsf{Inv}$ -> $\mathsf{GetResource}$ -> $\mathsf{Transaction}$.

> NOTE: $\mathsf{GetMempool}$ is currently never used in the reference implementation.


#### `GetBlocks`
This messages is sent to synchronize the blockchain with a peer. Its payload includes a $Locator$ block hash indicating the highest block known to the sender. 

This message is used in two situations:
 - if the block acceptance timeout expires (i.e., no new block is accepted for 20 seconds) <!-- TODO: define BLOCK_ACCEPTANCE_TIMEOUT -->
 - when the node starts a [synchronization procedure][syn] with a peer;

The response to this message should be an $\mathsf{Inv}$ message containing a vector of type $BlockFromHash$ with the list of all blocks in the [local chain][lc] that are successors of $Locator$.


| Field     | Type      | Size     | Description                         |
|-----------|-----------|----------|-------------------------------------|
| $Locator$ | SHA3 Hash | 32 bytes | The hash of the highest block known |

The expected message flow is the following: $\mathsf{GetBlocks}$ -> $\mathsf{Inv}$ -> $\mathsf{GetResource}$ -> $\mathsf{Block}$.

#### `GetResource`
This message is sent by a node to request a transaction or block.
The message can be used as a *direct request* to a peer, or to start a *resource discovery* process through the network.

The message has the following payload:

| Field       | Type          | Size            | Description                            |
|-------------|---------------|-----------------|----------------------------------------|
| $Inventory$ | [`Inv`][inv]  | 410-28448 bytes | List of resources being requested      |
| $Requester$ | SocketAddress | 410-28448 bytes | The IP address of the requesting node  |
| $TTL$       | Timestamp     | 8 bytes         | The expiration timeout                 |
| $HopsLimit$ | Unsigned Int  | 2 bytes         | Maximum number of hops for the request |

In response to this message, nodes send, in separate $\mathsf{Block}$ or $\mathsf{Transaction}$ messages, all available requested resources.

##### Resource Discovery
$\mathsf{GetResource}$ messages are propagated following a *Flood with Random Walk* algorithm. The algorithm works as follows:
- The requesting node sends the message to $RedundancyPeerCount$ random peers;
- When receiving a message, each node:
  - Checks if the message is expired ($TTL \gt \tau_{Now}$)
  - If not expired:
    - if the resources is known, it sends it to $Requester$
    - otherwise, it forwards the message, with $HopsLimit$ decreased by 1, to one random peer;

> Note: $TTL$ is currently not used in the reference implementation. By default, it is then set to its maximum value.

**Parameters**

| Name                  | Value | Description                                 |
|-----------------------|-------|---------------------------------------------|
| $RedundancyPeerCount$ | $8$   | Initial number of peers to send the request |
| $DefaultHopsLimit$    | $16$  | Default value for $HopsLimit$               |

##### Direct Request
When $\mathsf{GetResource}$ is sent in response to a $\mathsf{Inv}$ message, or to request the $Tip$'s successor as part of the [*PreSync*][ps] procedure, $HopsLimit$ is set to $1$.
This effectively makes $\mathsf{GetResource}$ a direct request to a specific peer.


#### `Inv`
This message is sent to advertise a list of known resources, such as transactions and blocks.
It is sent by a node in response to $\mathsf{GetMempool}$ and $\mathsf{GetBlocks}$ messages.

It's payload is an `Inv` structure containing the list of resources.

When receiving this message, nodes check if there are any unknown items and, if so, it requests them through a $\mathsf{GetResource}$ message with $HopsLimit = 1$.


<p><br></p>

## Procedures
While the underlying network protocol is described in the [Network][net] section, we here define some generic network procedures used in the SA protocol.

#### *Broadcast*
<!-- TODO: to how many peers? -->
This procedure is used by the creator of a new message to initiate the network propagation of the message. The function simply takes any consensus message in input and is assumed to not fail (only within the context of this description) so it does not return anything.

Note that, for the sake of readability, broadcast messages are also received by the local node (as in a self-sending). <!-- TODO: check if this is still necessary -->

#### *Receive*
This procedure is used to process incoming messages. Specifically, $\textit{Receive}(\mathsf{T},R,I)$ returns a message $\mathsf{M}$ of type $\mathsf{T}$ ($\mathsf{M^T}$) if it was received from the network and it has $Round_\mathsf{M}=R$ and $Iteration_\mathsf{M}=I$. 
If no new message has been received, it returns $NIL$.
When not relevant (e.g. with $\mathsf{Block}$ messages) $R$ and $I$ are ignored.

Messages received before calling the *Receive* function are stored in a queue and are returned by the function in the order they were received.

Note that also messages broadcasted by the node are received by this function. <!-- TODO: check if this is still necessary -->

#### *Propagate*
<!-- TODO: to how many peers? -->
This procedure represents a re-broadcast operation. It is used by a node when receiving a message from the network and propagating to other nodes.

We assume that if the message was originally broadcasted by the local node it is not re-propagated. <!-- TODO: check if this is still necessary -->

#### *Send*
This procedure represents a point-to-point message from the node to one of its peers. Its defined as *Send*$(\mathcal{P},\mathsf{M})$, where $\mathcal{P}$ is the recipient peer, and $\mathsf{M}$ is the message to send.


<!----------------------- FOOTNOTES ----------------------->

[^1]: Note that the Round field is not strictly necessary to identify a specific iteration and fork. However, we include it to simplify the identification of the corresponding round.

[^2]: Note that the BLS signature definition already includes the hashing of the message being signed. However, we explicitly show it here to make it clear and show the actual hash function used in our protocol (Blake2B).


<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md -->

[cmsgs]: #consensus
[msen]:  #sender

[cinf]:  #consensusinfo
[sigs]:  #signatures
[sinf]:  #signinfo

[cmsg]:  #candidate
[vmsg]:  #validation
[rmsg]:  #ratification
[qmsg]:  #quorum
[nmsg]:  #cmsg

[dx]:    #data-exchange
[inv]:   #inv
[ii]:    #invitem
[bmsg]:  #block
[txmsg]: #transaction
[gmmsg]: #getmempool
[gbmsg]: #getblocks
[grmsg]: #getresource
[rd]:    #resource-discovery
[dr]:    #direct-request
[imsg]:  #inv-1

[mx]:    #procedures-1
[broad]: #broadcast
[rec]:   #receive
[pmsg]:  #propagate
[send]:  #send

<!-- Notation -->
[hash]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/notation.md#hash-functions

<!-- Basics -->
[b]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#block-structure
[cb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#candidate-block
[fb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#full-block
[lc]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#local-chain
<!-- TODO -->
[tx]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#transaction


[atts]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestations
[att]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestation
[sv]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#stepvotes

<!-- Protocol -->
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md

[syn]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#synchronization
[ps]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#presync

<!-- Network -->
[net]:  https://github.com/dusk-network/dusk-protocol/tree/main/network/README.md


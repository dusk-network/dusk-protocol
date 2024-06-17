# Protocol Messages
This section describes the network messages exchanged by nodes to participate in the SA consensus protocol.

**ToC**
  - [Consensus Messages](#consensus-messages)
      - [Sender](#sender)
      - [`ConsensusInfo`](#consensusinfo)
      - [`VoteInfo`](#voteinfo)
    - [Signatures](#signatures)
      - [SignInfo](#signinfo)
    - [Messages](#messages)
      - [`Candidate`](#candidate)
      - [`Validation`](#validation)
      - [`Ratification`](#ratification)
      - [`Quorum`](#quorum)
      - [`Block`](#block)
      - [Sync messages](#sync-messages)
    - [Procedures](#procedures)
      - [*Msg*](#msg)
      - [*Broadcast*](#broadcast)
      - [*Receive*](#receive)
      - [*Propagate*](#propagate)
      - [*Send*](#send)


## Consensus Messages
To run the SA protocol, nodes exchange four main types of messages:
- `Candidate`: it stores a candidate block for a specific round and iteration; it is used during the [Proposal][prop] step;
- `Validation`: it contains the vote of a member of the voting committee for a [Validation][val] step;
- `Ratification`: it contains the vote of a member of the voting committee for a [Ratification][rat] step;
- `Quorum`: it contains the aggregated votes of a specific iteration's Validation and Ratification steps; it is generated at the end of an SA iteration if a quorum was reached in the Ratification step;

Consensus messages are composed of a [`ConsensusInfo`][cinf] structure, some fields specific to the message type, and, with the exception of the `Quorum` message, a [`SignInfo`][sinf] structure, containing the public key and signature of the provisioner that created the message. 

#### Sender
All messages include a $Sender$ field indicating the network identity (e.g. the IP address) of the peer from which the message was received. 
For the sake of readability, this field is omitted from the structure definition.


#### `ConsensusInfo`
All consensus messages share a common `ConsensusInfo` structure that allows identifying the candidate block they refer to. The information included is the previous block hash, and the round and iteration numbers.

Recall that there's a different candidate block for each round and iteration. However, in the case of a fork, two candidates could exist for the same round and iteration. Including the previous block's hash allows distinguishing the two candidates.

The `ConsensusInfo` structure is defined as follows:

| Field       | Type    | Size     | Description         |
|-------------|---------|----------|---------------------|
| $PrevHash$  | SHA3    | 32 bytes | Previous block hash |
| $Round$     | Integer | 64 bits  | Round number        |
| $Iteration$ | Integer | 64 bits  | Iteration number    |

The structure's total size is 48 bytes.

#### `VoteInfo`
This contains the information of a [Validation][val] or [Ratification][rat] vote.

| Field           | Type    | Size     | Description            |
|-----------------|---------|----------|------------------------|
| $Vote$          | Integer | 1 byte   | Validation vote        |
| $CandidateHash$ | SHA3    | 32 bytes | Candidate block's hash |

The structure's total size is 33 bytes.

$Vote$ can be $NoCandidate$ ($0$), $Valid$ ($1$), $Invalid$ ($2$), or $NoQuorum$ ($3$).
When $Vote$ is $NoCandidate$ or $NoQuorum$, $CandidateHash$ is empty.
Note that an empty $CandidateHash$ is not included in the [*signature value*][sigs].


### Signatures
Each message contains a *message signature* (included in the $Signature$ field) which is used to verify the message authenticity but is also functional to prove the reached agreement over a candidate block (see [Attestations][atts])

Formally, given a message $\mathsf{M}$, we define its signature $\eta_\mathsf{M}$ as

$$\eta_\mathsf{M} = Sign_{BLS}(Hash_{Blake2B}(\upsilon_\mathsf{M}), sk),$$

where $sk$ is the secret key paired with the message's $Signer$, and $\upsilon$ is the *signature value*, that is, the content of the message being signed[^1].
Note that the signature value depends on the message type.


***Procedures***
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

| Field           | Type                    | Size      | Description     |
|-----------------|-------------------------|-----------|-----------------|
| $ConsensusInfo$ | [`ConsensusInfo`][cinf] | 48 bytes  | Consensus info  |
| $Candidate$     | [`Block`][b]            |           | Candidate block |
| $SignInfo$      | [`SignInfo`][sinf]      | 144 bytes | Signature info  |

The message has a variable size of 192 bytes plus the block size.

The message's [signature value][sigs] $\upsilon_\mathcal{M^C}$ is:

$$\upsilon_\mathcal{M^C} = (ConsensusInfo || \eta_{Candidate})$$


#### `Validation`
This message is used by a member of a [Validation][val] committee to cast a vote on a candidate block. 

The message has the following structure:

| Field           | Type                    | Size      | Description     |
|-----------------|-------------------------|-----------|-----------------|
| $ConsensusInfo$ | [`ConsensusInfo`][cinf] | 48 bytes  | Consensus info  |
| $VoteInfo$      | [`VoteInfo`][vinf]      | 33 byte   | Validation vote |
| $SignInfo$      | [`SignInfo`][sinf]      | 144 bytes | Signature info  |

The message has a total size of 225 bytes.

In this message, $Vote$ (part of $VoteInfo$) can be $Valid$, $Invalid$, or $NoCandidate$.

The message's [signature value][sigs] $\upsilon_\mathcal{M^V}$ is:

$$\upsilon_\mathcal{M^V} = (ConsensusInfo || VoteInfo || ValStep)$$


#### `Ratification`
This message is used by a member of a [Ratification][rat] committee to cast a vote on a candidate block.

The message has the following structure:
| Field             | Type                    | Size      | Description                 |
|-------------------|-------------------------|-----------|-----------------------------|
| $ConsensusInfo$   | [`ConsensusInfo`][cinf] | 48 bytes  | Consensus info              |
| $VoteInfo$        | [`VoteInfo`][vinf]      | 33 byte   | Ratification vote           |
| $ValidationVotes$ | [`StepVotes`][sv]       | 56 byte   | Aggregated Validation votes |
| $Timestamp$       | Unsigned Integer        | 64 bits   | Timestamp in Unix format    |
| $SignInfo$        | [`SignInfo`][sinf]      | 144 bytes | Signature info              |

The message has a total size of 281 bytes.

In this message, $Vote$ (part of $VoteInfo$) can be $NoCandidate$, $Valid$, $Invalid$, or $NoQuorum$.
The $Timestamp$ field is reserved for a future feature.

The message's [signature value][sigs] $\upsilon_\mathcal{M^R}$ is:

$$\upsilon_\mathcal{M^R} = (ConsensusInfo || VoteInfo || RatStep)$$


#### `Quorum`
This message is used at the end of a successful SA iteration to communicate to the network that a quorum of votes has been reached. The message is generated by any node collecting a quorum of votes for Ratification. The payload includes the `Attestation` with votes from both [Validation][val] and [Ratification][rat] steps.

| Field           | Type                    | Size      | Description           |
|-----------------|-------------------------|-----------|-----------------------|
| $ConsensusInfo$ | [`ConsensusInfo`][cinf] | 48 bytes  | Consensus info        |
| $VoteInfo$      | [`VoteInfo`][vinf]      | 33 byte   | Quorum vote           |
| $Attestation$   | [`Attestation`][att]    | 112 bytes | Iteration attestation |

The message has a total size of 249 bytes.

Note that this message is not signed, since the $Attestation$ is already made of aggregated signatures, which can be verified against the $ConsensusInfo$ and the $VoteInfo$. As a consequence, any node of the network can broadcast this message, when collecting a quorum of Ratification votes.


#### `Block`
This message is used to propagate a winning block to other peers.
It contains a full block structure ([`Block`][b]) including an Attestation.

#### Sync messages
The following messages are used to synchronize the node with other peers:

- $\mathsf{GetBlocks}$: requests blocks from a peer;
- $\mathsf{Inv}$: advertises blocks or transactions by their hash
- $\mathsf{GetData}(T, data)$: requests blocks by their hash or height, or transactions by their hash
  - The payload type $T$ can be: $BlockFromHeight$, $BlockFromHash$, or $MempoolTx$
- $\mathsf{GetCandidate}$: requests a candidate block from its hash
- $\mathsf{GetCandidateResp}$: transmits a candidate block in response to  $\mathsf{GetCandidate}$ message;


### Procedures
While the underlying network protocol is described in the [Network][net] section, we here define some generic network procedures used in the SA protocol.

#### *Msg*
We define a common *Msg* procedure, which is used to create a consensus message:

$Msg(\mathsf{T}, f_1,\dots,f_2):$
1. $\mathsf{ConsensusInfo} = (\eta_{Tip}, R, I)$
2. $\sigma_{M^T} = Sign_{BLS}(\upsilon_\mathsf{M^T}, sk_\mathcal{N})$
3. $\mathsf{SignInfo} = (pk_\mathcal{N}, \sigma_{M^T})$
4. $\mathsf{M^T} = (\mathsf{ConsensusInfo}, f_1, \dots, f_n, SignInfo)$
5. $\texttt{output } \mathsf{M^T}$

The procedure uses the local consensus parameters ($Tip$, $R$, and $I$) and the node secret key $sk_\mathcal{N}$ to generate the message header and then builds the message with the other parameters $(f_1,\dots,f_2)$.

We assume the procedure is able to automatically generate $\upsilon_\mathsf{M^T}$ based on the message type and inputs.

$\mathsf{T}$ indicates the message type: $\mathsf{Candidate}$, $\mathsf{Validation}$, $\mathsf{Ratification}$, or $\mathsf{Quorum}$.

#### *Broadcast*
This procedure is used by the creator of a new message to initiate the network propagation of the message. The function simply takes any consensus message in input and is assumed to not fail (only within the context of this description) so it does not return anything.

Note that, for the sake of readability, broadcast messages are also received by the local node (as in a self-sending). <!-- TODO: check if this is still necessary -->

#### *Receive*
This procedure is used to process incoming messages. Specifically, $\textit{Receive}(\mathsf{T},R,I)$ returns a message $\mathsf{M}$ of type $\mathsf{T}$ ($\mathsf{M^T}$) if it was received from the network and it has $Round_\mathsf{M}=R$ and $Iteration_\mathsf{M}=I$. 
If no new message has been received, it returns $NIL$.
When not relevant (e.g. with $\mathsf{Block}$ messages) $R$ and $I$ are ignored.

Messages received before calling the *Receive* function are stored in a queue and are returned by the function in the order they were received.

Note that also messages broadcasted by the node are received by this function. <!-- TODO: check if this is still necessary -->

#### *Propagate*
This procedure represents a re-broadcast operation. It is used by a node when receiving a message from the network and propagating to other nodes.

We assume that if the message was originally broadcasted by the local node it is not re-propagated. <!-- TODO: check if this is still necessary -->

#### *Send*
This procedure represents a point-to-point message from the node to one of its peers. Its defined as *Send*$(\mathcal{P},\mathsf{M})$, where $\mathcal{P}$ is the recipient peer, and $\mathsf{M}$ is the message to send.


<!----------------------- FOOTNOTES ----------------------->

[^1]: Note that the BLS signature definition already includes the hashing of the message being signed. However, we explicitly show it here to make it clear and show the actual hash function used in our protocol (Blake2B).


<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md -->
[cinf]: #consensusinfo
[vinf]: #voteinfo
[sinf]: #signinfo
[sigs]: #signatures
[mx]:   #procedures
[msg]:  #msg
[sm]:   #sync-messages

[net]: https://github.com/dusk-network/dusk-protocol/tree/main/network/README.md

<!-- Blockchain -->
[b]:   https://github.com/dusk-network/dusk-protocol/tree/main/blockchain/README.md#block

<!-- Basics -->
[atts]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#attestations
[att]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#attestation
[sv]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#stepvotes

<!-- Consensus -->
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal/README.md
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md



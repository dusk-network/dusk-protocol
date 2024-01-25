<!-- TODO: mv Verify message functions here -->
# Messages
This section describes the network messages exchanged by nodes to participate in the SA consensus protocol.

## ToC
  - [Consensus Messages](#consensus-messages)
    - [Hash and Signature](#hash-and-signature)
    - [Structures](#structures)
      - [ConsensusHeader](#consensusheader)
      - [Candidate](#candidate)
      - [Validation](#validation)
      - [Ratification](#ratification)
      - [Quorum](#quorum)
      - [Block](#block)
    - [Procedures](#procedures)
      - [*Msg*](#msg)
      - [*Broadcast*](#broadcast)
      - [*Receive*](#receive)
      - [*Propagate*](#propagate)
      - [*Send*](#send)

## Consensus Messages
To run the SA protocol, nodes exchange four main types of messages:
- $\mathsf{Candidate}$: it stores a candidate block for a specific round and iteration; it is used during the [Proposal][prop] step;
- $\mathsf{Validation}$: it contains the vote of a member of the voting committee for a [Validation][val] step;
- $\mathsf{Ratification}$: it contains the vote of a member of the voting committee for a [Ratification][val] step;
- $\mathsf{Quorum}$: it contains the aggregated votes of a specific iteration's Validation and Ratification steps; it is generated at the end of an SA iteration if a quorum was reached in the Ratification step;

Formally, we denote a consensus message $\mathsf{M}$ as:

$`$\mathsf{M} = (\mathsf{H_M}, \sigma_\mathsf{M}, f_1,\dots,f_n),$`$

where $`\mathsf{H_M}`$ is the message header ([ConsensusHeader][ch]), $`\sigma_\mathsf{M}`$ is the provisioner signature on the header, and $f_1, \dots, f_n$ are the other fields specific to each message type.

### SAInfo
`SAInfo` includes information required to identify a candidate block: the previous block hash and the SA round and iteration.

Recall that there's a different candidate block for each round and iteration. However, in the case of a fork, two candidates could exist for the same round and iteration. Including the previous block's hash allows distinguishing the two candidates.

The `SAInfo` structure is: 
| Field       | Type          | Size      | Description                       |
|-------------|---------------|-----------|-----------------------------------|
| $PrevHash$  | SHA3          | 32 bytes  | Previous block's hash             |
| $Round$     | Integer       | 64 bits   | Round number                      |
| $Iteration$ | Integer       | 64 bits   | Iteration number                  |

The structure's total size is 48 bytes.

#### ConsensusHeader
All consensus messages share a common $\mathsf{ConsensusHeader}$ structure, defined as follows:

| Field       | Type              | Size      | Description                       |
|-------------|-------------------|-----------|-----------------------------------|
| $SAInfo$    | [`SAInfo`][#info] | 48 bytes  | Consensus info for the message    |
| $Signer$    | BLS Key           | 96 bytes  | Public Key of the message creator |
| $Signature$ | BLS Signature     | 48 bytes  | Message signature                 |

The $ConsensusHeader$ structure has a total size of 192 bytes.

**Signer**
The $Signer$ field identifies the provisioner sending the message. For instance, in a $Candidate$ message, this field identifies the block generator, while, in a $\mathsf{Validation}$ message, it identifies the committee member who cast the vote.


**Signature**
The $Signature$ field is not just the signature of the whole message but varies depending on the message type. This signature is used for the consensus protocol, especially for $\mathsf{Validation}$ and $\mathsf{Ratification}$ messages, whose signature is used to prove the vote and, if a quorum is reached, to certify the quorum in an iteration.

**Sender**
For all messages, we additionally define a $Sender$ field indicating the network identity (e.g. the IP address) of the peer from which the message was received. For the sake of simplicity, we omit this field from the structure definition.

### Messages Signatures
Each message contains a *message signature* (included in the $Signature$ field) which is used to verify the message content but is also functional to prove the reached agreement over a candidate block (see [Certificates][certs])

Formally, given a message $\mathsf{M}$, we define its signature $\eta_\mathsf{M}$ as

$$\eta_\mathsf{M} = Sign_{BLS}(Hash_{Blake2B}(\upsilon_\mathsf{M}), sk),$$

where $sk$ is the secret key paired with the message's $Signer$, and $\upsilon$ is the *signature value*, that is, the content of the message being signed[^1].
Note that the signature value depends on the message type.

***Procedures***
With respect to message signatures, we also define the following procedures:
 - $SignMessage(\mathsf{M})$: takes a message $\mathsf{M}$ and outputs the message signature $\sigma_\mathsf{M}$;
 - $Verify(\mathsf{M}, pk)$: takes a message $\mathsf{M}$ and a public key $pk$ and outputs $true$ if $\sigma_\mathsf{M}$ is a valid signature of $pk$ over the signature value $\upsilon_\mathsf{M}$.


### Structures

#### Candidate
The $\mathsf{Candidate}$ message is used by a block generator to broadcast a candidate block.

The message has the following structure:

| Field       | Type                  | Size      | Description           |
|-------------|-----------------------|-----------|-----------------------|
| $Header$    | [ConsensusHeader][ch] | 192 bytes | Message header        |
| $Candidate$ | [Block][b]            |           | Candidate block       |

The $\mathsf{Candidate}$'s *signature value* $\upsilon_\mathcal{M^C}$ is:

$$\upsilon_\mathcal{M^C} = (SAInfo || \eta_{Candidate})$$

The $\mathsf{Candidate}$ message has a variable size of 192 bytes plus the block size.

#### Validation
The $\mathsf{Validation}$ message is used by a member of a [Validation][val] committee to cast a vote on a candidate block. 

The message has the following structure:

| Field           | Type                  | Size      | Description            |
|-----------------|-----------------------|-----------|------------------------|
| $Header$        | [ConsensusHeader][ch] | 192 bytes | Message header         |
| $Vote$          | Integer               | 1 byte    | Validation vote        |
| $CandidateHash$ | SHA3                  | 32 bytes  | Candidate block's hash |

$Vote$ can be $Valid$, $Invalid$, or $NoCandidate$.

The $\mathsf{Validation}$'s *signature value* $\upsilon_\mathcal{M^V}$ is:

$$\upsilon_\mathcal{M^V} = (SAInfo || Vote || CandidateHash)$$


The $\mathsf{Validation}$ message has a total size of 225 bytes.

#### Ratification
The $\mathsf{Ratification}$ message is used by a member of a [Ratification][rat] committee to cast a vote on a candidate block. The vote is expressed by the $Header$'s $BlockHash$ field: if containing a hash, the vote is in favor of the corresponding block; if empty ($NIL$), the vote is against the candidate block of the $Round$ and $Iteration$ specified in the $Header$.


The message has the following structure:
| Field             | Type                  | Size      | Description                 |
|-------------------|-----------------------|-----------|-----------------------------|
| $Header$          | [ConsensusHeader][ch] | 192 bytes | Message header              |
| $Vote$            | Integer               | 1 byte    | Ratification vote           |
| $CandidateHash$   | SHA3                  | 32 bytes  | Candidate block's hash      |
| $ValidationVotes$ | [StepVotes][sv]       | 56 byte   | Aggregated Validation votes |

$Vote$ can be $Valid$, $Invalid$, $NoCandidate$, or $NoQuorum$.

The $\mathsf{Ratification}$'s *signature value* $\upsilon_\mathcal{M^R}$ is:

$$\upsilon_\mathcal{M^R} = (SAInfo || Vote || CandidateHash)$$

The $\mathsf{Ratification}$ message has a total size of 281 bytes.

#### Quorum
The $\mathsf{Quorum}$ message is used at the end of a successful SA iteration to communicate to the network that a quorum of votes has been reached. The message is generated by any node collecting a quorum of votes for Ratification. The payload includes the $Certificate$ with votes from both Validation and Ratification steps.

| Field         | Type                  | Size      | Description           |
|---------------|---------------------  |-----------|-----------------------|
| $Header$      | [ConsensusHeader][ch] | 192 bytes | Message header        |
| $Vote$        | Integer               | 1 byte    | Winning vote          |
| $Certificate$ | [Certificate][cert]   | 112 bytes | Iteration certificate |

Note that the $\mathsf{Quorum}$ message is not signed, since the $Certificate$ is already made of signatures, which can be verified against the $SAInfo$ and the $Vote$

The $\mathsf{Quorum}$ message has a total size of 249 bytes.

#### Block

The $\mathsf{Block}$ message is used to propagate a winning block to other peers.
This message only contains a full block structure ([Block][b]) including a Certificate.

#### Other messages
TBD
- $\mathsf{GetBlocks}$
- $\mathsf{Inv}$
- $\mathsf{GetData}$
- $\mathsf{GetCandidate}$
- $\mathsf{GetCandidateResp}$


### Procedures
While the underlying network protocol is described in the [Network][net] section, we here define some generic network procedures used in the SA protocol.

#### *Msg*
We define a common $Msg$ procedure, which is used to create a consensus message:

$Msg(\mathsf{T}, f_1,\dots,f_2):$
1. $`\mathsf{SAInfo} = (\eta_{Tip}, R, I)`$
2. $`\sigma_{M^T} = Sign_{BLS}(\upsilon_\mathsf{M^T}, sk_\mathcal{N})`$
3. $`\mathsf{H_M} = (\mathsf{SAInfo}, pk_\mathcal{\mathcal{N}}, \sigma_{M^T})`$
4. $`\mathsf{M^T} = (\mathsf{H_M}, f_1, \dots, f_n)`$
5. $\texttt{output } \mathsf{M^T}$

The procedure uses the local (to the node) consensus parameters ($Tip$, $R$, and $I$) and the node secret key $sk_\mathcal{N}$ to generate the message header and then builds the message with the other parameters $(f_1,\dots,f_2)$.

$\mathsf{T}$ indicates the message type: $\mathsf{Candidate}$, $\mathsf{Validation}$, $\mathsf{Ratification}$, or $\mathsf{Quorum}$.

#### *Broadcast*
The *Broadcast* function is used by the creator of a new message to initiate the network propagation of the message. The function simply takes any consensus message in input and is assumed to not fail (only within the context of this description) so it does not return anything.

Note that, for the sake of readability, broadcast messages are also received by the local node (as in a self-sending). <!-- TODO: check if this is still necessary -->

#### *Receive*
We use the *Receive* function to process incoming messages. Specifically, $\textit{Receive}(\mathsf{T},R,I)$ returns a message $\mathsf{M}$ of type $\mathsf{T}$ ($\mathsf{M^T}$) if it was received from the network and it has $Round_\mathsf{M}=R$ and $Iteration_\mathsf{M}=I$. 
If no new message has been received, it returns $NIL$.
When not relevant (e.g. with $\mathsf{Block}$ messages) $R$ and $I$ are ignored.

Messages received before calling the *Receive* function are stored in a queue and are returned by the function in the order they were received.

Note that also messages broadcasted by the node are received by this function. <!-- TODO: check if this is still necessary -->

#### *Propagate*
The *Propagate* function represents a re-broadcast operation. It is used by a node when receiving a message from the network and propagating to other nodes.

We assume that if the message was originally broadcasted by the local node it is not re-propagated. <!-- TODO: check if this is still necessary -->

#### *Send*
The *Send* function represents a point-to-point message from the node to one of its peers. Its defined as *Send*$(\mathcal{P},\mathsf{M})$, where $\mathcal{P}$ is the recipient peer, and $\mathsf{M}$ is the message to send.



[^1]: Note that the BLS signature definition already includes the hashing of the message being signed. However, we explicitly show it here to make it clear and show the actual hash function used in our protocol (Blake2B).


<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md -->
[ch]: #message-header
[mx]: #procedures
[#info]: #sainfo

<!-- Blockchain -->
[b]:   https://github.com/dusk-network/dusk-protocol/tree/main/blockchain/README.md#block

<!-- Basics -->
[certs]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#certificates
[cert]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#certificate
[sv]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md#stepvotes

<!-- Validation -->
[val]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md

<!-- Ratification -->
[rat]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md
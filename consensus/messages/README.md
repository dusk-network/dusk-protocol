# Messages
This section describes the network messages exchanged by nodes to participate in the Dusk consensus protocol.

## ToC
- [Consensus Messages](#consensus-messages)
  - [Message Header](#message-header)
  - [Message Hash and Signature](#message-hash-and-signature)
  - [Message Creation](#message-creation)
  - [Message Exchange](#message-exchange)
- [Message Structures](#message-structures)
  - [`NewBlock` Message](#newblock-message)
  - [`Reduction` Message](#reduction-message)
  - [`Agreement` Message](#agreement-message)
  - [`AggrAgreement` Message](#aggragreement-message)
  - [`Block` Message](#block-message)

## Consensus Messages
To run the SA protocol, participating nodes exchange consensus messages. There are four types of message:
- $\mathsf{NewBlock}$: this message stores a candidate block for a certain round and iteration. It is used during the [Attestation][att] phase;
- $\mathsf{Reduction}$: this message contains the vote of a provisioner, selected as a member of the voting committee, on the validity of a candidate block. It is used during the [Reduction][red] phase;
- $\mathsf{Agreement}$: this message contains the aggregated votes of a Reduction phase. It is used at the end of a Reduction phase, if a quorum is reached;
- $\mathsf{AggrAgreement}$: it contains an aggregation of $\mathsf{Agreement}$ messages. It is used in the [Ratification][rat] phase, if a quorum of such messages is received.

Formally, we denote a consensus message $\mathsf{M}$ as:

$`$\mathsf{M} = (\mathsf{H_M}, \sigma_\mathsf{M}, f_1,\dots,f_n),$`$

where $`\mathsf{H_M}`$ is the message header (defined below), $`\sigma_\mathsf{M}`$ is the provisioner signature on the header, and $f_1, \dots, f_n$ are the other fields specific to each message type.

In the following, we describe both the header and signature in detail.

### Message Header
<!-- TODO: Maybe we should define an SA Header (round,step,blockhash) as a separate structure, so we can add other fields to the message header, like the signer and the sender -->
All consensus messages share a common $MessageHeader$ structure, defined as follows:

| Field       | Type    | Size      | Description                      |
|-------------|---------|-----------|----------------------------------|
| $Signer$    | BLS Key | 96 bytes  | Public Key of the message signer |
| $Round$     | Integer | 64 bits   | Consensus round                  |
| $Step$      | Integer | 8 bits    | Consensus step                   |
| $BlockHash$ | Sha3    | 32 bytes  | Candidate block hash             |

$Signer$ is the public BLS key of the provisioner who creates and signs the message. Therefore, in a $NewBlock$ message, this field identifies the block generator, while, in a $\mathsf{Reduction}$ message, it identifies the committee member who casted the vote.

The $MessageHeader$ structure has a total size of 137 bytes.

### Message Hash and Signature
A *message signature* is the signature of the sender provisioner over the message header's hash.

Formally, given a message $\mathsf{M}$, we define its *hash* as:

$$\eta_\mathsf{M} = Hash_{Blake2B}(Round_\mathsf{M}||Step_\mathsf{M}||BlockHash_\mathsf{M}),$$

where $Round_\mathsf{M}$, $Step_\mathsf{M}$, and $BlockHash_\mathsf{M}$ are the respective fields included in $\mathsf{M}$'s header.

We then define the $\mathsf{M}$'s signature $\sigma_\mathsf{M}$ as:
$$\sigma_\mathsf{M} = Sign_{BLS}(\eta_\mathsf{M}, sk),$$
where $\eta_\mathsf{M}$ is the hash of $\mathsf{M}$'s header and $sk$ is the secret key of the message $Signer_\mathsf{M}$.

The signature $\sigma_\mathsf{M}$ is contained in the $Signature$ field of the message.

With respect to message signatures, we also define the following signing and verification functions:
 - $Sign(\mathsf{M}, sk)$, which takes a message $\mathsf{M}$, a secret key $sk$, and outputs the signature $\sigma_\mathsf{M}$;
 - $Verify(\mathsf{M})$, which takes a message $\mathsf{M}$ and outputs $true$ if $Signature = \sigma_{\mathsf{M}}$, and $false$ otherwise.
 

> Note that the hash operation is actually included in the definition of BLS signature. We explicitly show it here to make it clear and show the actual hash function used in our protocol (Blake2B).


### Message Creation
In addition, we define a common $Msg$ function, which is used to create a consensus message:

$Msg(\mathsf{Type}, f_1,\dots,f_2):$
1. $`\mathsf{H_M} = (pk_\mathcal{\mathcal{N}}, r, s, \mathsf{B}^c)`$
2. $`\sigma_{M} = Sign_{BLS}(\eta_\mathsf{M}, sk_\mathcal{N})`$
3. $`\mathsf{M}^\mathsf{Type} = (\mathsf{H_M}, \sigma_\mathsf{M}, f_1, \dots, f_n)`$
4. $\texttt{output } \mathsf{M}^\mathsf{Type}$

The function uses the local (to the node) consensus parameters and secret key to generate the message header and signature and then build the message with the other specific fields.

$\mathsf{Type}$ indicates the actual message ($\mathsf{NewBlock}$, $\mathsf{Reduction}$, or $\mathsf{Agreement}$).
In case of $\mathsf{Reduction}$, the parameter $f_1$, i.e. the vote $v$, is assigned to $BlockHash$ in the header.


### Message Exchange
While the underlying network protocol is described in the [Network][net] section, we here define some generic network functions that will be used in the consensus procedures.

***Broadcast***
The *Broadcast* function is used by the creator of a new message to initiate the network propagation of the message. The function simply takes any consensus message in input and is assumed to not fail (only within the context of this description) so it does not return anything.

***Receive***
<!-- TODO: define Receive for round r and iteration i -->
We use the *Receive* function to process incoming messages. Specifically, we define the *Receive*$(Type,r,s)$ function, which returns a message $\mathsf{M}$ of type $Type$ if it was received from the network and it has $Round_\mathsf{M}=r$ and $Step_\mathsf{M}=s$. If no new message has been received, it returns $NIL$.
When not relevant (e.g. with $\mathsf{Block}$ messages) $r$ and $s$ are ignored.

Messages received before calling the *Receive* function are stored in a queue and are returned by the function in the order they were received.

***Propagate***
The *Propagate* function represents a re-broadcast operation. It is used by a node when receiving a message from the network and propagating to other nodes.

***Send***
The *Send* function represents a point-to-point message from the node to one of its peers. Its defined as *Send*$(\mathcal{P},\mathsf{M})$, where $\mathcal{P}$ is the recipient peer, and $\mathsf{M}$ is the message to send.

***Sender***
In addition to exchange functions, we define a common $Sender$ field for all messages, which indicates the identity (e.g. the IP address) of the peer from which the message was received. 
For the sake of simplicity, we omit this field from the structure definition.

## Message Structures

### NewBlock Message
The $\mathsf{NewBlock}$ message is used by a block generator to broadcast a candidate block.

The message has the following structure:

| Field       | Type                  | Size      | Description           |
|-------------|-----------------------|-----------|-----------------------|
| $Header$    | [*MessageHeader*][mh] | 137 bytes | Message header        |
| $PrevHash$  | SHA3 Hash             | 256 bits  | Previous block's hash |
| $Candidate$ | [*Block*][b]          |           | Candidate block       |
| $Signature$ | BLS Signature         | 48 bytes  | Message signature     |

The $\mathsf{NewBlock}$ message has a variable size of 217 bytes plus the block size.

### Reduction Message
The $\mathsf{Reduction}$ message is used by a member of a Reduction committee to cast a vote on a candidate block. The vote is expressed by the $Header$'s $BlockHash$ field: if containing a hash, the vote is in favor of the corresponding block; if containing a $NIL$, the vote is against the candidate block of the $Round$ and $Step$ specified in the $Header$.


The message has the following structure:
| Field       | Type                  | Size      | Description           |
|-------------|-----------------------|-----------|-----------------------|
| $Header$    | [*MessageHeader*][mh] | 137 bytes | Message header        |
| $Signature$ | BLS Signature         | 48 bytes  | Signature of $Header$ |

The $\mathsf{Reduction}$ message has a total size of 185 bytes.

### Agreement Message
<!-- TODO: remove Signature -->
The $\mathsf{Agreement}$ message is used at the end of a successful SA iteration to communicate to the network that a quorum of (positive) votes has been reached. The message is generated by each node participating in the second Reduction committee that collects a quorum of votes for both reductions. The payload includes the $StepVotes$ from the Reduction phases.

| Field       | Type                | Size      | Description                      |
|-------------|---------------------|-----------|----------------------------------|
| $Header$    | [MessageHeader][mh] | 137 bytes | Consensus header                 |
| $Signature$ | BLS Signature       | 48 bytes  | Message signature                |
| $Certificate$    | [Certificate][cert][ ]  | 112 bytes | First and second Reduction votes |

The $\mathsf{Agreement}$ message has a total size of 297 bytes.

### Block Message

The $\mathsf{Block}$ message is used to propagate a winning block to other peers.
This message only contains a full block structure ([*Block*][b]) including a Certificate, plus the sender peer $\mathcal{S}$.

<!-- TODO 
- $\mathsf{GetBlocks}$
- $\mathsf{Inv}$
- $\mathsf{GetData}$
- $\mathsf{GetCandidate}$
- $\mathsf{GetCandidateResp}$
-->


<!------------------------- LINKS ------------------------->

[mh]: #message-header
[mx]: #message-exchange

<!-- Reduction -->
[sv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md#stepvotes

<!-- Ratification -->
[cert]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md#certificate
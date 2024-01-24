# Basics of SA Consensus

## Provisioners and Stakes
A provisioner is a user that locks a certain amount of their Dusk coins as *stake* (see [*Stake Contract*][c-stake]).
Formally, we define a *Provisioner* $\mathcal{P}$ as:

$$\mathcal{P}=(pk_\mathcal{P}, S_\mathcal{P}),$$

where $pk_\mathcal{P}$ is the BLS public key of the provisioner and $S_\mathcal{P}$ is the stake belonging to $\mathcal{P}$.

In turn, a *Stake* is defined as:
$$S_\mathcal{P}=(Amount, Height),$$
where $\mathcal{P}$ is the provisioner that owns the stake, $Amount$ is the quantity of locked Dusks, and $Height$ is the height of the block where the lock action took place (i.e., when the *stake* transaction was included). The minimum value for a stake is defined by the [global parameter][cp] $MinStake$, and is currently equivalent to 1000 Dusk.

The stake amount of each provisioner directly influences the probability of being extracted by the [Deterministic Sortition][dsa] algorithm: the higher the stake, the more the provisioner will be extracted on average.

### Epochs and Eligibility
Participation in the consensus protocol is marked by *epochs*. Each epoch corresponds to a fixed number of blocks, defined by the [global parameter][cp] $Epoch$ (currently equal to 2160).
<!-- TODO: why 2160 ? -->

The *Provisioners Set* during an epoch is fixed: all `stake` and `unstake` operations take effect at the beginning of an epoch. We refer to this property as *epoch stability*.

Additionally, for a Provisioner to be included in the Provisioners List, its stake is required to wait a certain number of blocks known as the *maturity* period. After this period, the stake and the Provisioner are said to be *eligible* for Sortition.

Formally, we say a Stake $S$ is *eligible* at round $R$, if $R$ is at least $Maturity$ blocks higher than the round in which $S$ was staked:

$$`R \ge Round_S + M,`$$

where $Round_S$ is the round at which $S$ was staked, and $M$ is the *maturity* period defined as:

$$`M = 2{\times}Epoch - (Round_S \mod Epoch)),`$$

where $Epoch$ is a [global parameter][cp]. Note that the value of $M$ is equal to a full epoch plus the blocks from $Round_S$ to the end of the corresponding epoch[^2]. Therefore the value of $M$ will vary depending on $Round_S$:

$$`Epoch \lt M \le 2{\times}Epoch.`$$

Having the Provisioner Set fixed during an epoch, along with the use of the maturity period, is mainly intended to allow the pre-verification of blocks from the future: if a node falls behind the main chain and receives a block at a higher height than its tip, it can pre-verify this block by ensuring that the block generator and the committee members are part of the expected Provisioner List.
Specifically, while the stability of the Provisioner List during an epoch allows to pre-verify blocks from the same epoch, the Maturity period allows to also pre-verify blocks from the next epoch, since all changes (stake/unstake) occurred between the tip and the end of the epoch will not be effective until the end of the following epoch.

### Candidate Block
A candidate block is the block generated in the [Proposal][prop] step by the provisioner extracted as block generator. This is the block on which other provisioners will have to reach an agreement. If an agreement is not reached by the end of the iteration, a new candidate block will be produced and a new iteration will start.

Therefore, for each iteration, only one (valid) candidate block can be produced[^1]. To reflect this, we denote a candidate block with $\mathsf{B}^c_{r,i}$, where $r$ is the consensus round, and $i$ is the consensus iteration. 
Note that we simplify this notation to simply $\mathsf{B}^c$ when this does not generate confusion.

A candidate block that reaches an agreement is called a *winning* block.


### Votes
<!-- TODO: move to Sortition/Voting Committees ? -->
<!-- DOING -->
In the Validation and Ratification steps, members of the voting committees cast their vote on the validity of the block, and to ratify the result of the Validation step.

Votes are in the form of BLS signatures, which allow them to be aggregated and verified together. This removes the necessity to store multiple signatures in the block or in the messages. Instead, a single aggregated signature, along with a *bitset* to indicate the signature of which committee members are included, is sufficient to validate the quorum on a candidate block.

In particular, each vote is the digital signature of the hash of the following fields: 
  - the previous block's hash, which identifies both the round and the branch to which the candidate is for;
  - the iteration number, to distinguish between votes for different candidates of the same round;
  - the candidate block's hash, to ensure which candidate the vote is for;
  - the step number, to distinguish Validation and Ratification votes.

## Certificates
<!-- TODO: Rename to Attestation and define Certificate as the PrevBlock Cert -->
A *Certificate* is an aggregated vote from a quorum committee along with the bitset of the voters. It is used as proof of a committee reaching Quorum or NilQuorum in the Reduction steps. 
In particular, Quorum Certificates are used to prove a candidate block reached consensus and can be accepted to the chain; on the other hand, NilQuorum Certificates are used to prove that a candidate failed to reach a quorum (and can't reach any).

### Structures
#### Certificate
The $\mathsf{Certificate}$ structure contains the aggregated votes of [sub-committees][sc] of the two [Reduction][rmsg] steps that reached quorum on a candidate block.
It is composed of two $\mathsf{StepVotes}$ structures, one for each [Reduction][red] step.


| Field             | Type            | Size     | Description                          |
|-------------------|-----------------|----------|--------------------------------------|
| $FirstReduction$  | [StepVotes][sv] | 56 bytes | Aggregated votes of first reduction  |
| $SecondReduction$ | [StepVotes][sv] | 56 bytes | Aggregated votes of second reduction |

The $\mathsf{Certificate}$ structure has a total size of 112 bytes.

#### StepVotes
The $\mathsf{StepVotes}$ structure is used to store votes from the [Validation][val] and [Ratification][rat] steps.
Votes are aggregated BLS signatures from members of a Voting Committee. Each vote is a signature for a triplet $(round, step, vote)$.
To specify which committee members' votes are included, a [sub-committee bitset][bs] is used.

The structure is defined as follows:

| Field    | Type          | Size     | Description                                |
|----------|---------------|----------|--------------------------------------------|
| $Voters$ | BitSet        | 64 bits  | Bitset of the voters                       |
| $Votes$  | BLS Signature | 48 bytes | Aggregated $\mathsf{Reduction}$ signatures |

The $\mathsf{StepVotes}$ structure has a total size of 56 bytes.

### Procedures
#### *AggregateVote*
*AggregateVote* adds a vote to a [StepVotes][sv] by aggregating the BLS signature and setting the signer bit in the committee bitset.

**Parameters**
- $\mathsf{SV}$: the $\mathsf{StepVotes}$ structure with the aggregated votes
- $\mathsf{C}$: the Voting Committee of $\mathsf{SV}$
- $\sigma$: the BLS signature to aggregate
- $pk$: the signer public key

**Procedure**

$\textit{AggregateVote}( \mathsf{SV}, \mathsf{C}, \sigma, pk ) :$
1. $\mathsf{SV}.Votes =$ *BLS_Aggregate*$(\mathsf{SV}.Votes, \sigma)$
2. $\mathsf{SV}.Voters = $ [*SetBit*][sb]$(\mathsf{SV}.Voters, \mathsf{C}, pk)$
3. $\texttt{output } \mathsf{SV}$
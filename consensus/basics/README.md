# Basics of SA Consensus
In this section, we describe the building blocks of the SA consensus protocol, such as *blocks*, *provisioners*, *voting committees*, and *attestations*.

### ToC
  - [Candidate Block](#candidate-block)
  - [Provisioners and Stakes](#provisioners-and-stakes)
    - [Epochs and Eligibility](#epochs-and-eligibility)
    - [Incentives and Penalties](#incentives-and-penalties)
      - [Rewards](#rewards)
      - [Slashing](#slashing)
  - [Voting Committees](#voting-committees)
    - [Step Committees](#step-committees)
    - [Votes](#votes)
    - [Step Numbers](#step-numbers)
    - [Block Generator Extraction](#block-generator-extraction)
    - [Subcommittees](#subcommittees)
      - [Bitsets](#bitsets)
    - [Procedures](#procedures)
      - [*ExtractGenerator*](#extractgenerator)
      - [*ExtractCommittee*](#extractcommittee)
      - [*BitSet*](#bitset)
      - [*SetBit*](#setbit)
      - [*CountSetBits*](#countsetbits)
      - [*SubCommittee*](#subcommittee)
      - [*CountCredits*](#countcredits)
  - [Attestations](#attestations)
    - [Structures](#structures)
      - [`Attestation`](#attestation)
      - [`StepVotes`](#stepvotes)
      - [`StepResult`](#stepresult)
    - [Procedures](#procedures-1)
      - [*AggregateVote*](#aggregatevote)


## Candidate Block
A candidate block is the block generated in the [Proposal][prop] step by the provisioner extracted as block generator. This is the block on which other provisioners will have to reach an agreement. If an agreement is not reached by the end of the iteration, a new candidate block will be produced and a new iteration will start.

Therefore, for each iteration, only one (valid) candidate block can be produced[^2]. To reflect this, we denote a candidate block with $\mathsf{B}^c_{R,I}$, where $R$ is the consensus round, and $i$ is the consensus iteration. 
Note that we simplify this notation to simply $\mathsf{B}^c$ when this does not generate confusion.

A candidate block that reaches an agreement is called a *winning* block.

## Provisioners and Stakes
A provisioner is a user that locks a certain amount of their Dusk coins as *stake* (see [*Stake Contract*][c-stake]).
Formally, we define a *Provisioner* $\mathcal{P}$ as:

$$\mathcal{P}=(pk_\mathcal{P}, S_\mathcal{P}),$$

where $pk_\mathcal{P}$ is the BLS public key of the provisioner and $S_\mathcal{P}$ is the stake belonging to $\mathcal{P}$.

In turn, a *Stake* is defined as:
$$S_\mathcal{P}=(Amount, Height),$$
where $\mathcal{P}$ is the provisioner that owns the stake, $Amount$ is the quantity of locked Dusks, and $Height$ is the height of the block where the lock action took place (i.e., when the *stake* transaction was included). The minimum value for a stake is defined by the [global parameter][cenv] $MinStake$, and is currently equivalent to 1000 Dusk.

The stake amount of each provisioner directly influences the probability of being extracted by the [Deterministic Sortition][ds] algorithm: the higher the stake, the more the provisioner will be extracted on average.

### Epochs and Eligibility
Participation in the consensus protocol is marked by *epochs*. Each epoch corresponds to a fixed number of blocks, defined by the [global parameter][cenv] $Epoch$ (currently equal to 2160).
<!-- TODO: why 2160 ? -->

The *Provisioners Set* during an epoch is fixed: all `stake` and `unstake` operations take effect at the beginning of an epoch. We refer to this property as *epoch stability*.

Additionally, for a Provisioner to be included in the Provisioners List, its stake is required to wait a certain number of blocks known as the *maturity* period. After this period, the stake and the Provisioner are said to be *eligible* for Sortition.

Formally, we say a Stake $S$ is *eligible* at round $R$, if $R$ is at least $Maturity$ blocks higher than the round in which $S$ was staked:

$$R \ge Round_S + M,$$

where $Round_S$ is the round at which $S$ was staked, and $M$ is the *maturity* period defined as:

$$M = 2{\times}Epoch - (Round_S \mod Epoch)),$$

where $Epoch$ is a [global parameter][cenv]. Note that the value of $M$ is equal to a full epoch plus the blocks from $Round_S$ to the end of the corresponding epoch[^1]. Therefore the value of $M$ will vary depending on $Round_S$:

$$Epoch \lt M \le 2{\times}Epoch.$$

Having the Provisioner Set fixed during an epoch, along with the use of the maturity period, is mainly intended to allow the pre-verification of blocks from the future: if a node falls behind the main chain and receives a block at a higher height than its tip, it can pre-verify this block by ensuring that the block generator and the committee members are part of the expected Provisioner List.
Specifically, while the stability of the Provisioner List during an epoch allows to pre-verify blocks from the same epoch, the Maturity period allows to also pre-verify blocks from the next epoch, since all changes (stake/unstake) occurred between the tip and the end of the epoch will not be effective until the end of the following epoch.

### Incentives and Penalties
Given the nature of the SA consensus protocol, it is of paramount importance that provisioners participate when selected. In fact, offline provisioners can slow down the network, and, in extreme cases, even stop block production. Specifically, the provisioner selected for the Proposal step must be online to produce the candidate block; similarly, a supermajority of voters is required to be online when the Validation or Ratification steps occur, to reach a quorum on the candidate block. If any of the two conditions are not met, the iteration fails and a new one is needed to create a new block.
To minimize the risk of failure, a number of incentives, in the form of *rewards*, and penalties, in the form of *slashing*, are included.

Many of these rewards and slashes are decided based on the block [attestations][atts], which are used as the source of truth when deciding on good or bad behaviors.
All operations are handled by the VM when performing the *state transition* of the accepted block.

#### Rewards 
The most basic reward is, for obvious reasons, the block reward, which is given to the generator of a candidate block that gets accepted into the chain.
This reward consists of two parts: the assignment of newly-emitted coins, and a portion of the fees of the transactions included in the block (for more details on fee distribution see the [Economic Protocol][ep]). In particular, the block generator is assigned 90% of the block emission (see the Dusk [emission schedule][tok]).

On the other hand, a small reward is also assigned to the voters of an accepted block[^3]. This not only incentivizes provisioners to stay online, but also disincentivizes skipping the vote at lower iterations. In fact, since the block producers of all iterations in the current round are known, if a block producer of a later iteration is in the voting committee of the current iteration, they might be incentivized to not vote in order to get to produce their block.
Additionally, we incentivize the block producer to include as many voters as possible by making a portion of the block reward proportional to the number of signatures in the block attestation. This is intended to disincentivize the block generator from cherry-picking signatures to penalize other provisioners.

Rewards are not added to the provisioner stake, but they are collected into a $Reward$ amount, from which the provisioner can later withdraw coins. This is done to limit the power-increasing effect of provisioners with bigger stakes being selected more (see [Deterministic Sortition][ds]).

#### Slashing
The following behaviors are subject to slashing:
- Missed block: if, in the [Proposal][prop] step, the selected generator fails to broadcast the candidate block; the slash takes effect when accepting a block with a [Failed Attestation][atts] of $NoCandidate$ quorum;
- Invalid block: if a candidate block is deemed invalid by the [Validation][val] step; the slash takes effect when accepting a block with a Failed Attestation of $Invalid$ quorum;
- Double voting: if a provisioner signs for two different votes regarding the same candidate block; the slash takes effect when an ad-hoc transaction is included in a block, containing the two conflicting signatures[^3];
- Double block: if a generator broadcasts two candidates for the same iteration; the slash takes effect when an ad-hoc transaction is included in a block, containing the two conflicting signatures.

Currently, the slash amount is equal to the block reward (which varies according to the [emission schedule][tok]).

Slashing is applied to the provisioner's reward first and then to the actual stake. That is, if the provisioner previously earned some rewards, then slashing affects this amount first. If the reward amount reaches zero, then slashing is applied to the actual stake.

Note that slashing has an immediate effect, in contrast with [staking][pro], which takes effect at the beginning of the second epoch after the stake operation.



## Voting Committees
A *Voting Committee* is an array of provisioners entitled to cast votes in the Validation and Ratification steps. Provisioners in a committee are called *members* of that committee. Each member in a given committee is assigned (by the sortition process) a number of *credits* (i.e., castable vote), referred to as its *power* in the committee.

Formally, a voting committee is defined as:

$`$ C_{R}^{S} = [m_0^C,\dots,m_n^C],$`$

where $R$ and $S$ are the consensus round and step, respectively, $n$ is the number of members in the committee, and each member $m_i^C$ is defined as:

$$ m_i^C = (pk_i, Power), $$ 

where $pk_i$ is the public key of the $i\text{th}$ provisioner to be added to the committee by the $DS$ algorithm, and $Power$ is the number of credits it is assigned.

Note that members are ordered by insertion (a provisioner is added to the committee when being assigned its first credit). That is, the first provisioner to be added to the list by $DS$ will have index $0$, and the last one will have index $n$.

For the sake of readability, in the rest of this documentation, we will use the following notation:
- $C[i]$ denotes the $i\text{th}$ member of committee $C$. Formally:
  - $C[i] = m_i^C$
- $i_m^C$ denotes the index $i$ of member $m$ in committee $C$. Formally:
  - $i_m^C = i : m_i^C = m$
- $m_{pk}^C$ denotes the member of $C$ with publick key $pk$.
  - $m_{pk}^C = m : m \in C \wedge pk_m = pk$
- We say a provisioner $P$ is in a committee $C$ if such committee contains a member with $P$'s public key. Formally:
  - $P \in C \text{ }if\text{ } \exists \text{ } m^C :   pk_{m^C}=pk_P$

### Step Committees
Voting Committees in the [Validation][val] and [Ratification][rat] steps have a fixed number of credits that is defined by the global [consensus parameter][cenv] $CommitteeCredits$, currently set to $64$.

When counting votes, each vote is multiplied by the power of the voter in the committee. For instance, if a member has 3 credits, his vote will be counted 3 times.

Hence, the $CommitteeCredits$ parameter determines the maximum number of members in a committee, and, indirectly, the degree of distribution of the voting process.

### Votes
In the Validation and Ratification steps, members of the voting committees cast their vote on the validity of the block, and to ratify the result of the Validation step.

Votes are in the form of BLS signatures, which allow them to be aggregated and verified together. This removes the necessity to store multiple signatures in the block or in the messages. Instead, a single aggregated signature, along with a *bitset* to indicate the signature of which committee members are included, is sufficient to validate the quorum on a candidate block.

In particular, each vote is the digital signature of the hash of the following fields: 
  - the previous block's hash, which identifies both the round and the branch to which the candidate is for;
  - the iteration number, to distinguish between votes for different candidates of the same round;
  - the step number, to distinguish Validation and Ratification votes.
  - the candidate block's hash, to ensure which candidate the vote is for;

### Step Numbers
In some contexts, it is useful to numerically identify the SA step.
For instance, this is done to distinguish Validation votes from Ratification votes, the step number (within the iteration) is included. 

In this respect, we define the following parameters:

| Parameter  | Value |
|------------|-------|
| $PropStep$ | 0     |
| $ValStep$  | 1     |
| $RatStep$  | 2     |

### Block Generator Extraction
To extract a block generator, a one-member committee is created, using the $Deterministic Sortition$ procedure with $credits = 1$, that is, by assigning a single credit. The provisioner being assigned such credit becomes the block generator for the iteration.

Formally, the block generator for round $R$ and step $S$ is defined as:

$$ \mathcal{G}_R^S = DS(R,S,1).$$

<p><br></p>

### Subcommittees
When votes of a committee reach a quorum they are aggregated into a single vote (i.e. a single, aggregated signature). The subset of the committee members whose vote is included in the aggregation is referred to as a *subcommittee*, or, if their votes reach a quorum, as a *quorum committee* (*q-committee* in short). 

To verify an aggregated vote, it is necessary therefore necessary to know the members of the corresponding subcommittee. This is achieved by means of *bitset*s. A bitset is simply a vector of bits that indicate for a given committee, which member is included and which not. 

For instance, in a committee $C=[\mathcal{P}_0,\mathcal{P}_1,\mathcal{P}_2,\mathcal{P}_3]$, a bitset $[0,1,0,1]$ would indicate the subcommittee including $\mathcal{P}_1$ and $\mathcal{P}_3$.

#### Bitsets
We make use of *bitsets* (array of bits) to indicate which members are part of a given subcommittee. 
Given a committee $\mathcal{C}$, a bitset indicates whether a member of $\mathcal{C}$ belongs to a subcommittee or not. 

In particular, if the *i*th bit is set (i.e., $i=1$), then the *i*th member of $\mathcal{C}$ is part of the subcommittee.

Note that a 64-bit bitset is enough to represent the maximum number of members in a committee (i.e., [*CommitteeCredits*][cenv]).

### Procedures

#### *ExtractGenerator*
This procedure extracts the block generator for the [Proposal][prop] step of round $R$ and iteration $I$ using the [*Deterministic Sortition*][ds] procedure.

***Parameters***
- $R$: round number
- $I$: iteration number

***Algorithm***

1. Get absolute step number $S$ for Proposal at iteration $I$
2. Execute Deterministic Sortition [*DS*][dsp] to extract the generator $\mathcal{G}$
3. Output $\mathcal{G}$

***Procedure***
$\textit{ExtractGenerator}(R,I)$
1. $S =$ [*GetStepNum*][gsn]$(I, PropStep)$
2. $\mathcal{G}=$ [*DS*][dsp]$(R, S, 1, Provisioners)$
3. $\texttt{output }\mathcal{G}$


#### *ExtractCommittee*
This procedure extracts the voting committee for the [Validation][val] or the [Ratification][rat] step of round $R$ and iteration $I$ using the [*Deterministic Sortition*][ds] procedure.

The procedure excludes the block generator $\mathcal{G}$ of the same iteration to prevent it from being part of the committee. In other words, a candidate generator cannot vote over its own block.

***Parameters***
- $R$: round number
- $I$: iteration number
- $StepNum$: the step number ($ValStep$ or $RatStep$)

***Algorithm***

1. Get absolute step number $S$ for $StepNum$ at iteration $I$
2. Extract iteration generator $\mathcal{G}$ for iteration $I$
3. Exclude the generator $\mathcal{G}$ from the provisioner list
4. Execute Deterministic Sortition [*DS*][dsp] to extract the committee $\mathcal{C}$
5. Output $\mathcal{C}$

***Procedure***
$\textit{ExtractCommittee}(R,I, StepNum)$
1. $S =$ [*GetStepNum*][gsn]$(I, StepNum)$
2. $\mathcal{G}_{R,I} =$ [*ExtractGenerator*][eg]$(R,I)$
3. $\boldsymbol{P} = Provisioners - \mathcal{G}_{R,I}$
4. $\mathcal{C}=$ [*DS*][dsp]$(R, S, CommitteeCredits, \boldsymbol{P})$
5. $\texttt{output } \mathcal{C}$


#### *BitSet*
This procedure takes a committee $C$ and list of provisioners $\boldsymbol{P}$, and outputs the bitset corresponding to the subcommittee of $C$ including provisioners in $\boldsymbol{P}$.

$\textit{BitSet}(C, \boldsymbol{P}=[pk_1,\dots,pk_n]) \rightarrow \boldsymbol{bs}_{\boldsymbol{P}}^C$

#### *SetBit*
This procedure sets a committee member's bit in a subcommittee bitset.

***Parameters***
- $\boldsymbol{bs}$: the subcommittee bitset
- $\mathcal{C}$: the committee
- $M$: the public key of the committee member

***Procedure***

 $\textit{SetBit}(\boldsymbol{bs}, \mathcal{C}, M)$
 - $\boldsymbol{bs}[i_M^\mathcal{C}] = 1$

#### *CountSetBits*
This procedure returns the amount of set bits in a bitset.

$\textit{CountSetBits}(\boldsymbol{bs}) \rightarrow setbits$

#### *SubCommittee*
This procedure takes a committee $C$ and a bitset $\boldsymbol{bs}^C$ and outputs the corresponding subcommittee.

$\textit{SubCommittee}(C, \boldsymbol{bs}^C) \rightarrow \boldsymbol{P}=[pk_1,\dots,pk_n]$

#### *CountCredits*
This procedure takes a committee $\mathcal{C}$ and a bitset $\boldsymbol{bs}$ and returns the cumulative amount of credits belonging to members of the subcommittee with respect to $\mathcal{C}$.

***Parameters***
- $\mathcal{C}$: a voting committee
- $\boldsymbol{bs}$: a subcommittee bitset 

***Procedure***

$\textit{CountCredits}(\mathcal{C}, \boldsymbol{bs}) \rightarrow credits$:
1. $\texttt{for } i=0 \dots CommitteeCredits{-}1 :$
   1. $\texttt{if } (\boldsymbol{bs}[i]=1):$
   2. $credits = credits + \mathcal{C}[i].Power$
2. $\texttt{output } credits$


## Attestations
An *Attestation* is an aggregate collection of votes from a specific iteration. It includes the votes of the [Validation][val] and [Ratification][rat] steps and is used as proof of a reached agreement in an iteration: if the iteration was successful, it proves a supermajority quorum of $Valid$ votes was cast in both steps, while if the iteration failed, it proves there was a majority quorum of $Invalid$, $NoCandidate$ or $NoQuorum$ votes. 
We use the terms *Valid Attestation* and *Failed Attestation* to refer to the two types of quorum being proved.

### Structures
#### `Attestation`
This structure contains the aggregated votes of the [Validation][val] and [Ratification][rat] steps of a single iteration. The votes of each step are contained in a [`StepVotes`][sv] structure.


| Field          | Type              | Size     | Description                               |
|----------------|-------------------|----------|-------------------------------------------|
| $Validation$   | [`StepVotes`][sv] | 56 bytes | Aggregated votes of the Validation step   |
| $Ratification$ | [`StepVotes`][sv] | 56 bytes | Aggregated votes of the Ratification step |

The structure has a total size of 112 bytes.

#### `StepVotes`
This structure is used to store votes from the [Validation][val] and [Ratification][rat] steps.
Votes are stored as the aggregated BLS signatures of the members of the two Voting Committees. In fact, each [vote][vot] is signed together with the related *Consensus Information* (round, iteration, step) and the hash of the candidate to which the vote refers.
To specify the votes from which committee members included in the aggregated signature, a [sub-committee bitset][bs] is used.

The structure is defined as follows:

| Field    | Type          | Size     | Description                |
|----------|---------------|----------|----------------------------|
| $Voters$ | BitSet        | 64 bits  | Bitset of the voters       |
| $Votes$  | BLS Signature | 48 bytes | Aggregated step signatures |

The structure has a total size of 56 bytes.

#### `StepResult`
This structure contains the result of a [Validation][val] or [Ratification][rat] step, that is the step result $Vote$ and the `StepVotes` with the aggregated votes producing $Vote$.

The structure is defined as follows:

| Field           | Type              | Size     | Description                  |
|---------------- |-------------------|----------|------------------------------|
| $Vote$          | Integer           | 8 bits   | The winning vote of the step |
| $CandidateHash$ | SHA3              | 32 bytes | The candidate hash           |
| $SV$            | [`StepVotes`][sv] | 56 bytes | Aggregated signatures        |

$Vote$ can be $Valid$, $Invalid$, $NoCandidate$, or $NoQuorum$.

### Procedures
#### *AggregateVote*
This procedure adds a vote to a [`StepVotes`][sv] by aggregating the BLS signature and setting the signer bit in the committee bitset.

**Parameters**
- $\mathsf{SV}$: the $\mathsf{StepVotes}$ structure with the aggregated votes
- $\mathcal{C}$: the Voting Committee of $\mathsf{SV}$
- $\sigma$: the BLS signature to aggregate
- $pk$: the signer public key

**Procedure**

$\textit{AggregateVote}( \mathsf{SV}, \mathcal{C}, \sigma, pk ) :$
1. $\mathsf{SV}.Votes =$ *BLS_Aggregate*$(\mathsf{SV}.Votes, \sigma)$
2. $\mathsf{SV}.Voters =$ [*SetBit*][sb]$(\mathsf{SV}.Voters, \mathcal{C}, pk)$
3. $\texttt{output } \mathsf{SV}$

<!-- TODO: Blockchain: Block/Transaction structures here ? -->


<!----------------------- FOOTNOTES ----------------------->

[^1]: Note that an epoch refers to a specific set of blocks and not just to a number of blocks; that is, an epoch starts and ends at specific block heights.

[^2]: In principle, a malicious block generator could create two valid candidate blocks. However, this case is automatically handled in the Validation step, since provisioners will reach agreement on a specific block hash.

[^3]: Note that rewards related to quorum voters are not yet implemented, so no specific reward amount has been yet decided. The same applies to slashing for double vote and double candidate.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md -->
[vot]:  #votes
[atts]: #attestations
[att]:  #attestation
[sv]:   #stepvotes
[sr]:   #stepresult
[av]:   #aggregatevote
[pro]:  #provisioners-and-stakes
[eg]:   #extractgenerator
[ec]:   #ExtractCommittee
[cc]:   #countcredits
[sc]:   #subcommittee
[cb]:   #countsetbits
[bs]:   #bitset
[sb]:   #setbit

<!-- Consensus -->
[cenv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#environment
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal/README.md
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md
[gsn]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#GetStepNum

<!-- Sortition -->
[ds]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md
[dsp]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#deterministic-sortition-ds 

<!-- TODO: link to Stake Contract -->
[c-stake]: https://github.com/dusk-network/dusk-protocol/tree/main/contracts/stake

[ep]: https://github.com/dusk-network/dusk-protocol/tree/main/economic-protocol
[tok]: https://docs.dusk.network/learn/economy/tokenomics/#token-emission-schedule
# Attestation
The attestation process in the SA protocol is the part that handles agreement of provisioners over blocks. In particular, the attestation of a block involves, at each round, selecting, among provisioners, a block generator, responsible for producing a candidate block, and two *voting committees*, responsible for verifying the block and reach agreement. 

When an agreement is reached on a candidate block, by a quorum of votes from both committees, the votes are collected, in aggregated form, into an *Attestation* structure, which serves as a proof of such an agreement.

**ToC**
  - [Voting Committees](#voting-committees)
    - [Step Committees](#step-committees)
    - [Step Numbers](#step-numbers)
    - [Block Generator Extraction](#block-generator-extraction)
    - [Procedures](#procedures)
      - [*ExtractGenerator*](#extractgenerator)
      - [*ExtractCommittee*](#extractcommittee)
    - [*GetQuorum*](#getquorum)
    - [*GetStepNum*](#getstepnum)
  - [Votes](#votes)
    - [`Vote`](#vote)
    - [Vote Signatures](#vote-signatures)
  - [Subcommittees](#subcommittees)
    - [Bitsets](#bitsets)
    - [Procedures](#procedures-1)
      - [*BitSet*](#bitset)
      - [*SetBit*](#setbit)
      - [*CountSetBits*](#countsetbits)
      - [*SubCommittee*](#subcommittee)
      - [*CountCredits*](#countcredits)
  - [Attestations](#attestations)
      - [Block Certificate](#block-certificate)
    - [Structures](#structures)
      - [`Attestation`](#attestation-1)
      - [`StepVotes`](#stepvotes)
      - [`StepResult`](#stepresult)
      - [`IterationResult`](#iterationresult)
    - [Procedures](#procedures-2)
      - [*AggregateVote*](#aggregatevote)
      - [*VerifyAttestation*](#verifyattestation)
      - [*VerifyVotes*](#verifyvotes)

<p><br></p>

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
The selection of the block generator is done throught the extraction of a one-member committee. Specifically, the $Deterministic Sortition$ procedure is executed to assign a single credit. The provisioner being assigned this credit becomes the block generator for the iteration.

Formally, the block generator ($\mathcal{G}$) for round $R$ and step $S$ is defined as:

$$ \mathcal{G}_R^S = DS(R,S,1).$$

### Procedures
#### *ExtractGenerator*
This procedure extracts the block generator for the [Proposal][prop] step of round $R$ and iteration $I$ using the [*Deterministic Sortition*][ds] procedure.

**Parameters**
- $R$: round number
- $I$: iteration number

**Algorithm**

1. Get absolute step number $S$ for Proposal at iteration $I$
2. Execute Deterministic Sortition [*DS*][dsp] to extract the generator $\mathcal{G}$
3. Output $\mathcal{G}$

**Procedure**
$\textit{ExtractGenerator}(R,I)$
1. $S =$ [*GetStepNum*][gsn]$(I, PropStep)$
2. $\mathcal{G}=$ [*DS*][dsp]$(R, S, 1, Provisioners)$
3. $\texttt{output }\mathcal{G}$


#### *ExtractCommittee*
This procedure extracts the voting committee for the [Validation][val] or the [Ratification][rat] step of round $R$ and iteration $I$ using the [*Deterministic Sortition*][ds] procedure.

The procedure excludes the block generator $\mathcal{G}$ of the same iteration to prevent it from being part of the committee. In other words, a candidate generator cannot vote over its own block.

The next-iteration generator is also excluded to mitigate the [Future-Iteration Generator Problem][fgi].

**Parameters**
- $R$: round number
- $I$: iteration number
- $StepNum$: the step number ($ValStep$ or $RatStep$)

**Algorithm**

1. Get absolute step number $S$ for $StepNum$ at iteration $I$
2. Extract iteration generator $\mathcal{G}_{R,I}$ for iteration $I$
3. Exclude $\mathcal{G}_{R,I}$ from the provisioner list
4. Extract iteration generator $\mathcal{G}_{R,I+1}$ for iteration $I+1$
5. Exclude $\mathcal{G}_{R,I+1}$ from the provisioner list
6. Execute Deterministic Sortition [*DS*][dsp] to extract the committee $\mathcal{C}$
7. Output $\mathcal{C}$

**Procedure**
$\textit{ExtractCommittee}(R,I, StepNum)$
1. $S =$ [*GetStepNum*][gsn]$(I, StepNum)$
2. $\mathcal{G}_{R,I} =$ [*ExtractGenerator*][eg]$(R,I)$
3. $\boldsymbol{P} = Provisioners - \mathcal{G}_{R,I}$
4. $\mathcal{G}_{R,I+1} =$ [*ExtractGenerator*][eg]$(R,I+1)$
5. $\boldsymbol{P} = Provisioners - \mathcal{G}_{R,I+1}$
6. $\mathcal{C}=$ [*DS*][dsp]$(R, S, CommitteeCredits, \boldsymbol{P})$
7. $\texttt{output } \mathcal{C}$


### *GetQuorum*
This procedure returns the quorum target depending on the vote $\mathsf{V}$

**Parameters**
- $\mathsf{V}$: the vote type ($Valid$, $Invalid$, $NoCandidate$, $NoQuorum$)

**Procedure**

$\textit{GetQuorum}(\mathsf{V}):$
- $\texttt{if } (\mathsf{V} = Valid): \texttt{output } Supermajority$
- $\texttt{else}: \texttt{output } Majority$

### *GetStepNum*
This procedure returns the absolute step number within the round. It is used for the [*DS*][ds] procedure.

**Parameters**
- $I$: the iteration number
- $StepNum$: the relative step number ($PropStep$, $ValStep$, $RatStep$)

**Procedure**

$\textit{GetStepNum}(I, Step):$
- $\texttt{output } I \times + StepNum$ 

<p><br></p>

## Votes
In the [Validation][val] and [Ratification][rat] steps, members of the [voting committees][vc] cast their vote on the validity of the block, and to ratify the result of the Validation step.

Votes are in the form of enumerations, whose data includes the hash of the candidate block in case of $Valid$ and $Invalid$ votes.

### `Vote`
This enum contains the information of a [Validation][val] or [Ratification][rat] vote.

It can have the following values:

| Variant             | Data   | Size     | Description                                    |
|---------------------|--------|----------|------------------------------------------------|
| $NoCandidate$ ($0$) |    /   | 1 byte   | Vote for missing candidate                     |
| $Valid$ ($1$)       | $SHA3$ | 33 bytes | Vote for valid candidate with specified hash   |
| $Invalid$ ($2$)     | $SHA3$ | 33 bytes | Vote for invalid candidate with specified hash |
| $NoQuorum$ ($3$)    |    /   | 1 byte   | Vote for no quorum reached                     |

The enum size is 33 bytes.

### Vote Signatures
To ensure authentication and integrity, votes are [digitally signed][sigs] in the corresponding [`Validation`][vmsg] and [`Ratification`][rmsg] messages.

Thanks to the use of BLS signatures, equal votes from different provisioners can be aggregated into a single signature. The aggregated vote can than be propagated, stored, and verified with the corresponding aggregated public keys of the voters. This efficient scheme is implemented in the [`StepVotes`][sv] structure, which is based on the concept of [subcommittees][subc].

<p><br></p>

## Subcommittees
When votes of a committee reach a quorum they are aggregated into a single vote (i.e. a single, aggregated signature). The subset of the committee members whose vote is included in the aggregation is referred to as a *subcommittee*, or, if their votes reach a quorum, as a *quorum committee* (*q-committee* in short). 

To verify an aggregated vote, it is necessary therefore necessary to know the members of the corresponding subcommittee. This is achieved by means of *bitset*s. A bitset is simply a vector of bits that indicate for a given committee, which member is included and which not. 

For instance, in a committee $C=[\mathcal{P}_0,\mathcal{P}_1,\mathcal{P}_2,\mathcal{P}_3]$, a bitset $[0,1,0,1]$ would indicate the subcommittee including $\mathcal{P}_1$ and $\mathcal{P}_3$.

### Bitsets
We make use of *bitsets* (array of bits) to indicate which members are part of a given subcommittee. 
Given a committee $\mathcal{C}$, a bitset indicates whether a member of $\mathcal{C}$ belongs to a subcommittee or not. 

In particular, if the *i*th bit is set (i.e., $i=1$), then the *i*th member of $\mathcal{C}$ is part of the subcommittee.

Note that a 64-bit bitset is enough to represent the maximum number of members in a committee (i.e., [*CommitteeCredits*][cenv]).

### Procedures

#### *BitSet*
This procedure takes a committee $C$ and list of provisioners $\boldsymbol{P}$, and outputs the bitset corresponding to the subcommittee of $C$ including provisioners in $\boldsymbol{P}$.

$\textit{BitSet}(C, \boldsymbol{P}=[pk_1,\dots,pk_n]) \rightarrow \boldsymbol{bs}_{\boldsymbol{P}}^C$

#### *SetBit*
This procedure sets a committee member's bit in a subcommittee bitset.

**Parameters**
- $\boldsymbol{bs}$: the subcommittee bitset
- $\mathcal{C}$: the committee
- $M$: the public key of the committee member

**Procedure**

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

**Parameters**
- $\mathcal{C}$: a voting committee
- $\boldsymbol{bs}$: a subcommittee bitset 

**Procedure**

$\textit{CountCredits}(\mathcal{C}, \boldsymbol{bs}) \rightarrow credits$:
1. $\texttt{for } i=0 \dots CommitteeCredits{-}1 :$
   1. $\texttt{if } (\boldsymbol{bs}[i]=1):$
   2. $credits = credits + \mathcal{C}[i].Power$
2. $\texttt{output } credits$


<p><br></p>


## Attestations
An *attestation* is a proof of a reached agreement for a specific iteration. It includes the quorum-reaching vote along with the aggregated signatures of the quorum committees from the [Validation][val] and [Ratification][rat] steps
As such, an attestation proves a either supermajority of $Valid$ votes was cast in both steps, if the iteration succeeded, or that there was a majority of $Invalid$, $NoCandidate$ or $NoQuorum$ votes, if the iteration failed. 
We use the terms *Success Attestation* and *Fail Attestation* to refer to the two types of results being proved.

#### Block Certificate
Note that, for each [candidate block][cb], there might be multiple Success Attestations, one for each possible subset of quorum voters. This might generate ambiguity in the handling of quorum-specific mechanisms, such as the assignment of [rewards][rew].

To make such mechanisms deterministic, the official set of voters that brought to a Success Attestation is decided by the generator of the following block.
This is done by including in the candidate an attestation of the previous block. Such an attestation is referred to as the *certificate* of the previous block.

### Structures
We define the following Attestation-related structures: 
  - [`Attestation`][att]: it contains a quorum of votes for the Validation and Ratification steps of a single iteration, in aggregated form;
  - [`StepVotes`][sv]: it contains votes for a specific step, in aggregated form;
  - [`StepResult`][sr]: it contains the winning vote of a step (the one reaching the quorum), and the corresponding signatures, in aggregated form;

#### `Attestation`
This structure contains the result of an iteration ([`IterationResult`][ir]) along with the aggregated signatures ([`StepVotes`][sv]) of the [Validation][val] and [Ratification][rat] steps.

| Field          | Type                    | Size     | Description                               |
|----------------|-------------------------|----------|-------------------------------------------|
| $Result$       | [`IterationResult`][ir] | 40 bytes | Result and vote of the iteration          |
| $Validation$   | [`StepVotes`][sv]       | 56 bytes | Aggregated votes of the Validation step   |
| $Ratification$ | [`StepVotes`][sv]       | 56 bytes | Aggregated votes of the Ratification step |

The structure has a total size of 152 bytes.

#### `StepVotes`
This structure is used to store votes in the [Validation][val] and [Ratification][rat] steps.
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

| Field  | Type              | Size     | Description                  |
|--------|-------------------|----------|------------------------------|
| $Vote$ | [`Vote`][vote]    | 33 bytes | The winning vote of the step |
| $SV$   | [`StepVotes`][sv] | 56 bytes | Aggregated signatures        |

$Vote$ can be $Valid$, $Invalid$, $NoCandidate$, or $NoQuorum$.

This structure has a total size of 89 bytes.

#### `IterationResult`
This enumeration contains the final result of an iteration, which can be $Success$ if a quorum of $Valid$ votes has been reached, or $Fail$, if any other quorum has been reached. 
Note that if no quorum has been reached, no Attestation can be created.


| Variant   | Data           | Size     | Description                                                                             |
|-----------|----------------|----------|-----------------------------------------------------------------------------------------|
| $Success$ | [`Vote`][vote] | 33 bytes | It represents a successful iteration; `Vote` can only be $Valid$                        |
| $Fail$    | [`Vote`][vote] | 33 bytes | It represents a failed iteration; `Vote` can be `NoCandidate`, `NoQuorum`, or `Invalid` |

This enumeration's size is 40 bytes


### Procedures
We define the following Attestation-related procedures: 
  - [*AggregateVote*][av]: adds a vote to a [`StepVotes`][sv]
  - [*VerifyAttestation*][va]: verifies an [`Attestation`][att]
  - [*VerifyVotes*][vv]: verifies aggregated votes in a [`StepVotes`][sv] and checks the quorum

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


#### *VerifyAttestation*
This procedure checks an Attestation by verifying the [Validation][val] and [Ratification][rat] aggregated signatures of the Vote against the respective committees.
It takes an optional $ExpectedResult$ against which to verify the Attestation's $Result$.

**Parameters**
- $\mathsf{CI}$: the [`ConsensusInfo`][cinf] for the Attestation
- $\mathsf{A}$: the [Attestation][att] to verify
- $ExpectedResult$: the expected result of the attestation ($Success$ or $Fail$)

**Algorithm**
1. Check the $Result$ against the $ExpectedResult$
2. Check both Validation and Ratification votes are present
3. Verify Validation votes
4. If votes are not valid, output $false$
5. Verify Ratification votes
6. If votes are not valid, output $false$
7. Output $true$

**Procedure**

$\textit{VerifyAttestation}(\mathsf{CI}, \mathsf{A}, ExpectedResult):$
- $\texttt{set}:$
   - $\eta_{\mathsf{B}}^p, R, I \leftarrow \mathsf{CI}$
   - $`\mathsf{IR}, \mathsf{SV}^V, \mathsf{SV}^R \leftarrow \mathsf{A}`$
   - $\mathcal{C}^V =$ [*ExtractCommittee*][ec]$(R,I, ValStep)$
   - $\mathcal{C}^R =$ [*ExtractCommittee*][ec]$(R,I, RatStep)$
   - $\mathsf{V}: \mathsf{IR}.Vote$
   - $\upsilon^V = (\mathsf{CI}||\mathsf{V}||ValStep)$ 
   - $\upsilon^R = (\mathsf{CI}||\mathsf{V}||RatStep)$
   - $Q =$ [*GetQuorum*][gq]$(\mathsf{V})$

1. $\texttt{if } (ExpectedResult \ne NIL) \texttt{ and } (\mathsf{IR} \ne ExpectedResult): \texttt{output } false$
2. $\texttt{if } (\mathsf{SV}^V = NIL) \texttt{ or } (\mathsf{SV}^R = NIL): \texttt{output } false$

3. $isValid =$ [*VerifyVotes*][vv]$`(\mathsf{SV}^V, \upsilon^V, Q, \mathcal{C}^V)`$
4. $\texttt{if } (isValid{=}false): \texttt{output } false$

5. $isValid =$ [*VerifyVotes*][vv]$`(\mathsf{SV}^R, \upsilon^R, Q, \mathcal{C}^R)`$
6. $\texttt{if } (isValid{=}false): \texttt{output } false$

7. $\texttt{output } true$

#### *VerifyVotes*
This procedure checks the aggregated votes are valid and reach the target quorum.

**Parameters**
- $\mathsf{SV}$: $\mathsf{StepVotes}$ with the aggregated votes
- $\upsilon$: the [signature value][sigs]
- $Q$: the target quorum
- $\mathcal{C}$: the step committee

**Algorithm**
1. Compute subcommittee $C^{\boldsymbol{bs}}$ from $\mathsf{SV}.BitSet$
2. If credits in $C^{\boldsymbol{bs}}$ are less than the target quorum $Q$
   1. Output $false$
3. Aggregate public keys of $C^{\boldsymbol{bs}}$ members
4. Verify aggregated signature over $\upsilon$

**Procedure**

$\textit{VerifyVotes}(\mathsf{SV}, \upsilon, Q)$:
- $\texttt{set}:$
  - $\boldsymbol{bs}, \sigma_{\boldsymbol{bs}} \leftarrow \mathsf{SV}$
1. $\mathcal{C}^{\boldsymbol{bs}}=$ [*SubCommittee*][sc]$(\mathcal{C}, \boldsymbol{bs})$
2. $\texttt{if } ($[*CountCredits*][cc]$(\mathcal{C}, \boldsymbol{bs}) \lt Q):$
   1. $\texttt{output } false$
3. $pk_{\boldsymbol{bs}} = AggregatePKs(C^{\boldsymbol{bs}})$
4. $\texttt{output } Verify_{BLS}(\upsilon, pk_{\boldsymbol{bs}}, \sigma_{\boldsymbol{bs}})$


<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md -->
[vc]:    #voting-committees
[sc]:    #step-committees
[sn]:    #step-numbers
[bge]:   #block-generator-extraction
[eg]:    #ExtractGenerator
[ec]:    #ExtractCommittee
[gq]:    #GetQuorum
[gsn]:   #GetStepNum

[vot]:   #votes
[vote]:  #vote
[vsigs]: #vote-signatures

[subc]:  #subcommittees
[bits]:  #bitsets
[bs]:    #bitset
[sb]:    #setbit
[csb]:   #countsetbits
[sc]:    #subcommittee
[cc]:    #countcredits

[atts]:  #attestations
[cert]:  #block-certificate
[att]:   #attestation
[sv]:    #stepvotes
[sr]:    #stepresult
[ir]:    #iterationresult
[av]:    #aggregatevote
[va]:    #verifyattestation
[vv]:    #verifyvotes


<!-- Notation -->
[hash]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/notation.md#hash-functions

<!-- Basics -->
[rew]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#rewards
[fgi]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#future-generator-incentive-problem

[cb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#candidate-block

<!-- Protocol -->
[cenv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#environment
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md
[ds]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md
[dsp]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md#deterministic-sortition-ds

[cinf]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#consensusinfo

<!-- Messages -->
[sigs]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#signatures
[vmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#validation
[rmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#ratification
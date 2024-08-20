# Succinct Attestation
**Succinct Attestation** (**SA**) is a permissionless, committee-based Proof-of-Stake consensus protocol. 
The protocol is run by Dusk stakers, known as ***provisioners***, which are responsible for generating and validating new blocks.

Provisioners participate in turns, based on the [*Deterministic Sortition*][ds] (*DS* in short) algorithm, which is used to select a unique *block generator* and unique *voting committees* among provisioners for each new block, in a decentralized, non-interactive way.

Before reading, please make sure you are familiar with the system [basics][bas] and the document [notation][not].

**ToC**
  - [Overview](#overview)
    - [Environment](#environment)
    - [Procedures](#procedures)
      - [*SAInit*](#sainit)
      - [*SALoop*](#saloop)
      - [*SARound*](#saround)
      - [*SAIteration*](#saiteration)
  - [Step Timeouts](#step-timeouts)
    - [Adaptive Timeout](#adaptive-timeout)
    - [Design](#design)
    - [Environment](#environment-1)
    - [Procedures](#procedures-1)
      - [*SetRoundTimeouts*](#setroundtimeouts)
      - [*StoreElapsedTime*](#storeelapsedtime)
      - [*AdjustBaseTimeout*](#adjustbasetimeout)
      - [*IncreaseTimeout*](#increasetimeout)
  - [Relaxed Mode](#relaxed-mode)
  - [Emergency Mode](#emergency-mode)
    - [Emergency Block](#emergency-block)
    - [Procedures](#procedures-2)
      - [*isEmergencyBlock*](#isemergencyblock)



## Overview
The SA protocol proceeds in ***rounds***, with each round adding a new block to the chain.
In turn, each round proceeds in ***iterations***, with each iteration aiming at generating a candidate block and reaching agreement among provisioners.

Each iteration is composed of three phases, or ***steps***:
  1. [***Proposal***][prop]: in this step, a *generator*, extracted via [*DS*][ds], creates a new candidate block and broadcasts it to the network using a `Candidate` message;
  
  2. [***Validation***][val]: in this step, a committee of provisioners, extracted via [*DS*][ds], votes on the validity of the candidate block via `Validation` messages. The committee can reach a successful *quorum* with a supermajority ($\frac{2}{3}$) of $Valid$ votes or an unsuccessful quorum with a majority ($\frac{1}{2}+1$) of $Invalid$ votes, cast for an invalid candidate, or $NoCandidate$ votes, cast when the candidate is unknown.
  
  3. [***Ratification***][rat]: in this step, a new committee of provisioners, extracted via [*DS*][ds], votes on the result of the previous Validation step via `Ratification` messages. If a quorum was reached they cast the winning Validation vote, otherwise they will vote $NoQuorum$. If a quorum is reached, a `Quorum` message is broadcast, containing an `Attestation` with the votes of the two steps.
  Similar to Validation, the step succeeds if a supermajority quorum of $Valid$ votes is received, and it fails with a majority quorum of $Invalid$, $NoCandidate$, or $NoQuorum$ votes.
  If no quorum is reached, the step result is unknown.

If a quorum of $Valid$ votes is reached in both the Validation and Ratification steps, the candidate block generated in the Proposal step is accepted as the new tip of the chain.
Conversely, if the Ratification result is a failure or unknown, a new iteration is executed (that is a new candidate block is produced and voted upon).

The round terminates when an iteration is successful.

The maximum number of iterations executed within a single round is determined by the global parameter $MaxIterations$.

### Environment
We here define global parameters and state variables of the SA protocol, used and shared by all consensus procedures.

The SA environment is composed of:
- *global parameters*: network-wide parameters used by all nodes of the network using a particular protocol version;
- *chain state*: represents the current system state, as per result of the execution of all transactions in the blockchain;
- *round state*: local variables used to handle the consensus state 

Additionally, we denote the node running the protocol with $\mathcal{N}$ and refer to its provisioner[^1] keys as $sk_\mathcal{N}$ and $pk_\mathcal{N}$.

**Global Parameters**
All global values (except for the genesis block) refer to version $0$ of the protocol.

| Name               | Value                                    | Description                                                              |
|--------------------|------------------------------------------|--------------------------------------------------------------------------|
| $Version$          | 0                                        | Protocol version number                                                  |
| $GenesisBlock$     | $\mathsf{B}_0$                           | Genesis block of the network                                             |
| $Dusk$             | 1000000000                               | Value of one unit of Dusk (in lux)                                       |
| $MinStake$         | 1000                                     | Minimum amount of a single stake (in Dusk)                               |
| $BlockGas$         | 5.000.000.000                            | Gas limit for a single block                                             |
| $Epoch$            | 2160                                     | Epoch duration in number of blocks                                       |
| $MinBlockTime$     | 10                                       | Minimum time between blocks (in seconds)                                 |
| $TimestampMargin$  | 3                                        | Margin of error when checking timestamps against local time (in seconds) |
| $CommitteeCredits$ | 64                                       | Total credits in a voting committee                                      |
| $Supermajority$    | $CommitteeCredits \times \frac{2}{3}$    | Supermajority quorum (43 credits)                                        |
| $Majority$         | $CommitteeCredits \times \frac{1}{2} +1$ | Majority quorum (33 credits)                                             |
| $MaxIterations$    | 50                                       | Maximum number of iterations in a single round                           |
| $RelaxedMode$      | 8                                        | Iteration after which the Relaxed mode is enabled                        |
| $EmergencyMode$    | $16$                                     | Iteration at which [Emergency Mode][em] is enabled                       |
| $DuskKey$          | $pk_\mathcal{Dusk}$                      | A Dusk-owned public key used to sign the [Emergency Block][eb]           |


**Chain State**
| Name                 | Type                   | Description                            |
|----------------------|------------------------|----------------------------------------|
| $\textbf{Chain}$     | [`ChainBlock`][chb][ ] | The [local chain][lc] of the node      |
| $Tip$                | [`Block`][b]           | The local chain tip (last block)       |
| $SystemState$        | [`VMState`][vms]       | Current [system state][sys]            |
| $Provisioners$       | [*Provisioner*][pro]   | Current set of (eligible) provisioners |

**Round State**
| Name                              | Type                    | Description                       |
|-----------------------------------|-------------------------| ----------------------------------|
| $Round$                           | Int                     | Current round number              |
| $Iteration$                       | Int                     | Current iteration number          |
| $\mathsf{B}^c$                    | [`Block`][b]            | Candidate block                   |
| $\mathsf{B}^w$                    | [`Block`][b]            | Winning block                     |
| $\boldsymbol{FailedIterations}$   | [`Attestation`][att][ ] | Attestations of failed iterations |


### Procedures
The SA consensus is defined by the [*SAInit*][init] procedure, which executes an infinite loop ([*SALoop*][sal]) of rounds ([*SARound*][sar]), each executing one or more iterations ([*SAIteration*][sai]) until a *winning block* ($\mathsf{B}^w$) is produced for the round, becoming the new $Tip$ of the chain ([*AcceptBlock*][ab]).
The consensus loop could be interrupted when receiving a valid [`Block`][bmsg] message (see [*HandleBlock*][hb]) which could trigger the [*fallback*][fal] or [*synchronization*][syn] procedures.
Similarly, receiving a [`Quorum`][qmsg] message could interrupt a consensus round by accepting a candidate as the new $Tip$ (see [*HandleQuorum*][hq]).


#### *SAInit*
This procedure is the entry point of a consensus node. 
Upon boot, the node checks if there is a local state saved and, if so, loads it. Otherwise, it sets the local $Tip$ to *GenesisBlock*. 
Then, it probes the network to check if it is in sync or not with the main chain. If not, it starts a synchronization procedure. 

When the node is synchronized, it starts [*SALoop*][sal] to execute the consensus *rounds* and [*HandleBlock*][hb] to handle incoming [`Block`][bmsg] messages. 

**Algorithm**

1. Load local state $S$
2. If there is no saved state
   1. Set $Tip$ to $GenesisBlock$
3. Otherwise
   1. Check $S$'s validity
   2. Set $SystemState$ to $S$
   3. Set $Tip$ to last block
4. Start SA loop ([*SALoop*][sal])
5. Start Block message handler ([*HandleBlock*][hb])

**Procedure**

$\textit{SAInit}():$
1. $S =$ *LoadState*$()$
2. $\texttt{if } (S = NIL):$
   1. $Tip = GenesisBlock$
3. $\texttt{else}:$
   1. *ValidateState*$(S)$    <!-- TODO -->
   2. $SystemState = S.SystemState$
   3. $Tip = S.Tip$
4. $\texttt{start}$([*SALoop*][sal])
5. $\texttt{start}$([*HandleBlock*][hb])

<p><br></p>

#### *SALoop*
This procedure executes an infinite loop of consensus rounds ([*SARound*][sar]). 
It is initially started by [*SAInit*][init] but it can be stopped and restarted due to [fallback][fal] or [synchronization][syn].

**Algorithm**

1. Loop:
   1. Set $Round$ to $Tip$'s height plus one
   2. Execute Round $Round$ to produce winning block $\mathsf{B}^w$

**Procedure**

$\textit{SALoop}():$
1. $\texttt{loop}:$
   1. $Round = Tip.Height + 1$
   2. [*SARound*][sar]$(Round)$

<p><br></p>

#### *SARound*
This procedure executes a single consensus round. First, it initializes the [*Round State*][cenv] variables; then, it starts the [*HandleQuorum*][hq] process in the background, to handle [`Quorum`][qmsg] messages for the round, and starts executing consensus iterations ([*SAIteration*][sai]). 
If, at any time, a winning block $\mathsf{B}^w$ is produced, as the result of a successful iteration or due to a `Quorum` message, it is accepted to the [local chain][lc] and the round ends. 
If, for any reason, the round ends without a winning block, the node halts the consensus loop ([*SALoop*][sal]) and waits for an Emergency Block signed by Dusk. A Dusk node would instead produce and broadcast an Emergency Block for the round.

**Algorithm**

1. Set candidate block $\mathsf{B}^c$ and winning block $\mathsf{B}^w$ to $NIL$
2. Reset $\boldsymbol{FailedIterations}$
3. Adjust step base timeouts ([*SetRoundTimeouts*][srt])
4. Start $\mathsf{Quorum}$ message handler ([*HandleQuorum*][hq])
5. For $Iteration$ from 0 to $MaxIterations$
   1. If in Emergency Mode:
      1. Start [*SAIteration*][sai] as a thread
      2. Wait $MaxStepTimeout$ for each step
   2. If not in Emergency Mode:
      1. Execute SA iteration ([*SAIteration*][sai])
   3. If a winning block $\mathsf{B}^w$ has been produced 
      1. Broadcast $\mathsf{B}^w$
      2. Accept $\mathsf{B}^w$ into the chain
      3. End round
6. If we reached $MaxIterations$ without a winning block
   1. Create an Emergency Block Request message
   2. Broadcast the message

**Procedure**

$\textit{SARound}():$
1. $\texttt{set }$:
   - $\mathsf{B}^c, \mathsf{B}^w = NIL$
2. $\boldsymbol{FailedIterations} = []$
3. [*SetRoundTimeouts*][srt]$()$
4. $\texttt{start}($[*HandleQuorum*][hq]$(Round))$
5. $\texttt{for } Iteration = 0 \dots MaxIterations-1 :$
   1. $\texttt{if } (I \ge EmergencyMode):$
      1. $\texttt{start}($[*SAIteration*][sai]$(Round, Iteration))$
      2. $\texttt{wait} (3 \times MaxStepTimeout)$
   2. $\texttt{else}:$
      1. [*SAIteration*][sai]$(Round, Iteration)$
   3. $\texttt{if }(\mathsf{B}^w \ne NIL):$
      1. [*Broadcast*][mx]$(\mathsf{B}^w)$
      2. [*AcceptBlock*][ab]$(\mathsf{B}^w)$
      3. $\texttt{break}$
6. $\texttt{if } (\mathsf{B}^w = NIL)$
   1. $\mathsf{M} =$ [*CMsg*][nmsg]$(EmergencyBlockRequest)$
   2. [*Broadcast*][mx]$(\mathsf{M})$

<p><br></p>

#### *SAIteration*
This procedure executes the sequence of [*Proposal*][prop], [*Validation*][val], and [*Ratification*][rat] steps.
The *Proposal* outputs the candidate block $\mathsf{B}^c$ for the iteration; this is passed to *Validation*, which outputs the step result with the quorum-reaching vote or $NoQuorum$ if the timeout expired; the Validation's result is then passed to the Ratification step, which, also outputs the step result. Step results are in the form of [`StepResult`][sr] structures, which contain the quorum-reaching [`Vote`][vote] and the aggregated signature of the quorum committee (or $NIL$ is the vote is $NoQuorum$).

If a quorum was reached in both Validation and Ratification, a [`Quorum`][qmsg] message is broadcast with the [`Attestation`][atts] of the iteration (i.e. the winning vote, and the two [`StepVotes`][sv] with the aggregated signatures of the quorum committee).

**Algorithm**
1. Run *Proposal* to generate the *candidate* block $\mathsf{B}^c$
2. Run *Validation* on $\mathsf{B}^c$
3. Run *Ratification* on the Validation result
4. If Ratification reached a quorum on $\mathsf{V}$: 
   1. If $\mathsf{V}$ is $Valid$
      1. Set the iteration result $Result$ to $Success$
   2. Otherwise set $Result$ to $Fail$
   3. Create an attestation $\mathsf{A}$ with the $Result$ and the Validation and Ratification step votes
   4. Create $\mathsf{Quorum}$ message $\mathsf{M^Q}$
   5. Broadcast $\mathsf{M^Q}$
   6. If the Ratification result is $Success$:
      1. Make $\mathsf{B}^c$ the winning block [*MakeWinning*][mw]
   7. If the Ratification result is $Fail$
      1. Add $\mathsf{A}$ to the $\boldsymbol{FailedIterations}$ list

**Procedure**
$\textit{SAIteration}(R, I):$
1. $\mathsf{B}^c =$ [*ProposalStep*][props]$(R, I)$
2. $\mathsf{SR}^V =$ [*ValidationStep*][vs]$(R, I, \mathsf{B}^c)$
3. $\mathsf{SR}^R =$ [*RatificationStep*][rs]$(R, I, \mathsf{SR}^V)$
- $\texttt{set}:$
  - $`\_, \mathsf{SV}^V \leftarrow \mathsf{SR}^V`$
  - $\mathsf{V}, \mathsf{SV}^R \leftarrow \mathsf{SR}^R$
1. $\texttt{if } (\mathsf{V} \ne NoQuorum):$
   1. $\texttt{if } \mathsf{V} = Valid$:
      1. $\texttt{set} Result = Success$
   2. $\texttt{else}:$
      1. $\texttt{set} Result = Fail$
   3. $\mathsf{A} = ({Result, \mathsf{SV}^V, \mathsf{SV}^R})$
   4. $\mathsf{M} =$ [*CMsg*][nmsg]$(\mathsf{Quorum}, \mathsf{A})$
      | Field           | Value                 |
      |-----------------|-----------------------|
      | $ConsensusInfo$ | $(\eta_{Tip}, R, I)$  |
      | $Attestation$   | $\mathsf{A}$          |
   5. [*Broadcast*][mx]$(\mathsf{M})$
   6. $\texttt{if } (Result = Success):$
      1. [*MakeWinning*][mw]$(\mathsf{B}^c, \mathsf{A})$
   7. $\texttt{else}:$
      1. $\boldsymbol{FailedIterations}[I] = {\mathsf{A}}$

<p><br></p>

## Step Timeouts
Each step of the [SA iteration][sai] has a limited timeout, within which the step can succeed. If the timeout expires, the step fails.

Step timeouts serve various purposes. First of all, they are used to keep provisioners aligned with each other: to reach consensus on a candidate block, provisioners need to run the same round and iteration at approximately the same time. Timeouts force the consensus loop to run on a fast pace, without indefinite waits for steps to end. For instance, if the provisioner node gets isolated from the network, it won't get stuck at the same round, but will move on until either an iteration succeeds or a full block is received.

Timeouts are also functional to avoid forks. Without timeouts, a step could run indefinitely until some result is obtained (e.g., a candidate block is received, or a quorum of votes is reached). Dealing with this would require iterations to run in parallel, leading to blocks that could be replaced at any moment by lower-iteration ones. Instead, by having a limited time on each step, provisioners always act in a finite time, allowing iterations to move on. For instance, if the candidate block does not arrive in time, the provisioner votes $NoCandidate$ in the Validation step. Similarly, if the Validation step does not reach any quorum within the timeout, the provisioner votes $NoQuorum$. Moreover, once a vote is cast, no other vote is allowed from the same provisioner (this behavior will be [penalized][pen] in the future). These timeout-induced votes enable blocks that are not produced at the first iteration to be [finalized][rf]. This is because a quorum of negative votes can be used to produce a [Fail Attestation][atts] that proves the iteration failed and can't reach any valid quorum.

Another effect of timeouts is enabling [punishing][pen] a provisioner for not producing a block. While timeouts cannot ensure a block was not produced at all (which is impossible, since the generator could create the block and then postpone its broadcast indefinitely) they allow provisioners to agree on the fact that a majority did not receive such a block.

### Adaptive Timeout
To cope with changes in the network latency, step timeouts are designed to self-adjust based on previous events.

Two mechanisms determine the timeout for a specific step execution:
  - expiration increase: if a timeout expires, it is increased by a fixed amount of time ($TimeoutIncrease$) for the next iteration; in other words, if the step failed (locally) because not enough time was given to receive all messages, the timeout is increased to allow such messages to be received in the next iteration.
  <!-- If our timeout is too low, and other provisioners reach consensus, we would receive the Quorum/Block message before succeeding. We should consider this for the next round.  -->
  - round adjustment: at each round, the timeout for the first iteration is given by the average of the past rounds; this mechanism allows to adjust the timeout to the actual time needed by the step: if the last executions of the step succeeded within X seconds, it means X seconds are sufficient for the step to complete. This also allows the timeout to be reduced if the network latency decreases.

### Design

There are three timeouts, one for each step:
- $\tau_{Prop}$, for $Proposal$
- $\tau_{Val}$, for $Validation$
- $\tau_{Rat}$, for $Ratification$

When a new round begins, each step's base timeout ($BaseTimeout_{Step}$) is adjusted to the rounded average of the past successful executions (up to $MaxElapsedTimes$ values); if no previous elapsed times are known, the base timeout is set to $MaxStepTimeout$.

At Iteration 0, each step timeout $\tau_{Step}$ is set to $BaseTimeout_{Step}$. Then, while executing iterations, if the step is successful, its elapsed time is stored in $ElapsedTimes_{Step}$; if the timeout expires, it is increased by $TimeoutIncrease$ for the next iteration.

### Environment

**Parameters**

| Name              | Value | Description                                               |
|-------------------|-------|-----------------------------------------------------------|
| $TimeoutIncrease$ | 2     | Increase amount in case of timeout (in seconds)           |
| $MinStepTimeout$  | 7     | Minimum timeout for a single step (in seconds)            |
| $MaxStepTimeout$  | 40    | Maximum timeout for a single step (in seconds)            |
| $MaxElapsedTimes$ | 5     | Maximum number of elapsed time values stored (in seconds) |

**State Variables**

| Name                  | Description                         |
|-----------------------|-------------------------------------|
| $\tau_{Step}$         | Current $Step$ timeout              |
| $BaseTimeout_{Step}$  | Current $Step$ base timeout         |
| $ElapsedTimes_{Step}$ | Stored $Step$ timeout elapsed times |

With $Step = Proposal, Validation, Ratification$

### Procedures

#### *SetRoundTimeouts*
This procedure sets the initial step timeout for all steps

$\textit{SetRoundTimeouts}()$
- $\texttt{for } Step \texttt{ in } [Proposal, Validation, Ratification] :$
  - [*AdjustBaseTimeout*][abt]$(Step)$
  - $\tau_{Step} = BaseTimeout_{Step}$

#### *StoreElapsedTime*
This procedure adds a new elapsed time to $ElapsedTimes_{Step}$. 
$ElapsedTimes$ storage is a queue structure with a max capacity of $MaxElapsedTimes$. As such, if there are $MaxElapsedTimes$ values stored, when a new value is inserted, the oldest one is removed.

$\textit{StoreElapsedTime}(Step, \tau_{Elapsed})$
- $\texttt{if } (|ElapsedTimes_{Step}| = MaxElapsedTimes):$
  - $ElapsedTimes_{Step}.pop()$
- $ElapsedTimes_{Step}.push(\tau_{Elapsed})$

#### *AdjustBaseTimeout*
This procedure adjusts the first-iteration timeout for step $Step$ based on the stored elapsed time values.
The new timeout is the rounded-up average value of the stored elapsed times.

**Parameters**
- $Step$: the step name

**Procedure**

$\textit{AdjustBaseTimeout}(Step):$
1. $\texttt{if } (|ElapsedTimes_{Step}| = 0):$
   1. $BaseTimeout_{Step} = MaxStepTimeout$ 
2. $\texttt{else}:$
   1. $AvgElapsed =$ *Ceil*$($*Sum*$(ElapsedTimes_{Step}) / |ElapsedTimes_{Step}|)$
   2. $BaseTimeout_{Step} =$ *Max*$(AvgElapsed, MinStepTimeout)$ 

#### *IncreaseTimeout*
This procedure increases a step timeout by $TimeoutIncrease$ seconds up to $MaxStepTimeout$.

**Parameters**
- $Step$: the step timeout

**Procedure**

$\textit{IncreaseTimeout}(Step):$
- $\tau_{Step} =$ *Max*$(\tau_{Step} + TimeoutIncrease, MaxStepTimeout)$

<p><br></p>


## Relaxed Mode
To mitigate the increase of the block size when having many failed iterations in one round, we limit the number of iterations for which to include a [Fail Attestation][atts] in the candidate block (see the $FailedIterations$ field in the [BlockHeader][bh] structure). In particular, candidate blocks only include Fail Attestations for the first $RelaxedMode$ iterations (see [Global Parameters][cenv]).

## Emergency Mode
In extreme cases, where most provisioners are offline or isolated, multiple consecutive iterations may fail due to the absence of the appointed generators or voters. In such a situation, the maximum number of iterations for a round may be reached. To avoid ending a round without an accepted block, the SA protocol can switch to the *Emergency Mode*.

This mode activates upon reaching iteration $EmergencyMode$ and essentially consists in disabling the [step timeouts][tim]. 
Without timeouts, iterations in Emergency Mode, or *emergency iterations*, do not terminate until a candidate block is generated and a quorum is reached in both [Validation][val] and [Ratification][rat]. In particular, an emergency iteration only moves to the next step if the previous step succeeds. As a consequence, $NoCandidate$ and $NoQuorum$ votes are disabled in Emergency Mode.
We refer to an active emergency iteration as an *open iteration*.

In Emergency Mode, new iterations are started after the maximum iteration time ($MaxStepTimeout \times 3$) elapsed. However, since iterations only terminates when a quorum is reached on a candidate, it is possible to have multiple open iterations at the same time.

This increases the possibility of producing a block for the round, but also the probability of forks. In fact, each of the running iterations could reach agreement on a candidate block, possibly generating multiple winning blocks for the same round. Nonetheless, forks are still automatically resolved by always choosing the lowest-iteration block (see [Chain Management][cm]).
Note that, if a block is accepted for the round, all open iterations are terminated. This decreases the chances of other candidate blocks reaching quorum.

When the last iteration ($MaxIterations$) is reached, all open iterations keep executing indefinitely, until either a quorum is reached on a candidate, or an [Emergency Block][eb] is received.

### Emergency Block
When the maximum iteration time ($MaxStepTimeout \times 3$) of the last iteration has elapsed, nodes broadcast an *Emergency Block Request* (EBR) to the network. This message is used to request special Dusk-owned nodes to produce an *Emergency Block*.

In particular, if such a request is received for a majority of the total stake in the network, the Emergency Block is produced.
This block has no transactions, but contains a new seed signed by Dusk (and verifiable with the [global parameter][cenv] $DuskKey$).
Additionally, a proof of the EBRs is included as the aggregated signatures on such messages.

If a node receives an Emergency Block, it accepts it into its local chain and moves to the next round. Nevertheless, the Emergency Block is marked as $Accepted$ (see [Consensus State][cs]), so it can be replaced by a lower-iteration block for the same round (unless it gets finalized due to [Rolling Finality][rf]).

Note that multiple Dusk nodes can generate the same block (with the exception of the EBRs proof, which is not included in the block hash), since it contains no transactions and is signed with the same key. Thus, no fork can be produced by multiple Emergency Blocks for the same round.

Also note that an Emergency Block is only accepted when the last iteration maximum time has elapsed.

### Procedures

#### *isEmergencyBlock*
This procedure outputs $true$ if the input block $\mathsf{B}$ is a valid Emergency Block.

$`\textit{isEmergencyBlock}(\mathsf{B})`$
- $\texttt{if} (\mathsf{B}.Iteration = MaxIterations)$
- $\texttt{and} (\mathsf{B}.Transactions = NIL)$
- $\texttt{and} (\mathsf{B}.Faults = NIL)$
- $\texttt{and} (\mathsf{B}.Generator = DuskKey):$
  - $\texttt{output} true$


<!----------------------- FOOTNOTES ----------------------->

[^1]: For the sake of simplicity, we assume the node handles a single provisioner identity.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md -->
[sa]:   #overview

[rm]:   #relaxed-mode
[em]:   #emergency-mode
[eb]:   #emergency-block
[ieb]:  #isemergencyblock

[cenv]: #environment
[init]: #sainit
[sal]:  #saloop
[sar]:  #saround
[sai]:  #saiteration

[tim]:  #step-timeouts
[tenv]: #environment-1
[set]:  #StoreElapsedTime
[srt]:  #SetRoundTimeouts
[abt]:  #adjustbasetimeout
[it]:   #increasetimeout


<!-- Basics -->
[bas]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md

[b]:     https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#block-structure
[bh]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#blockheader-structure
[lc]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#local-chain
[chb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#chainblock-structure
[sys]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#system-state
[vms]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#vmstate-structure
[cs]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#consensus-state
[rf]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#rolling-finality

[pro]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#provisioners-and-stakes
[pen]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#penalties

[gq]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetQuorum
[gsn]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetStepNum
[atts]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestations
[att]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestation
[vote]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#vote
[sc]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#subcommittee
[sb]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#setbit
[sv]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#stepvotes
[sr]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#stepresult

[not]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/notation.md

<!-- Protocol -->
[prop]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md
[props]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md#Proposalstep

[val]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md
[vs]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md#ValidationStep

[rat]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md
[rs]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md#RatificationStep

[ds]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md

[cm]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md
[syn]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#synchronization
[ab]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#acceptblock
[hb]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#handleblock
[hq]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#handlequorum
[mw]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#makewinning
[fal]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#fallback
[syn]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/chain-management.md#synchronization

<!-- Messages -->
[nmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#cmsg
[qmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#quorum
[bmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#block
[mx]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#procedures-1

<!-- Network -->
[net]:   https://github.com/dusk-network/dusk-protocol/tree/main/network/README.md
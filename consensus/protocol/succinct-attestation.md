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
  - [Emergency Mode](#emergency-mode)
    - [Emergency Block](#emergency-block)
    - [Procedures](#procedures-2)
      - [*BroadcastEmergencyBlock*](#broadcastemergencyblock)
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

A maximum number of 255 iterations is executed within a single round.

### Environment
We here define global parameters and state variables of the SA protocol, used and shared by all consensus procedures.

The SA environment is composed of:
- *global parameters*: network-wide parameters used by all nodes of the network using a particular protocol version;
- *chain state*: represents the current system state, as per result of the execution of all transactions in the blockchain;
- *round state*: local variables used to handle the consensus state 

Additionally, we denote the node running the protocol with $\mathcal{N}$ and refer to its provisioner[^1] keys as $sk_\mathcal{N}$ and $pk_\mathcal{N}$.

**Global Parameters**
All global values (except for the genesis block) refer to version $0$ of the protocol.

| Name               | Value                                    | Description                                                    |
|--------------------|------------------------------------------|----------------------------------------------------------------|
| $Version$          | 0                                        | Protocol version number                                        |
| $GenesisBlock$     | $\mathsf{B}_0$                           | Genesis block of the network                                   |
| $Dusk$             | 1000000000                               | Value of one unit of Dusk (in lux)                             |
| $BlockGas$         | 5.000.000.000                            | Gas limit for a single block                                   |
| $MinStake$         | 1000                                     | Minimum amount of a single stake (in Dusk)                     |
| $Epoch$            | 2160                                     | Epoch duration in number of blocks                             |
| $CommitteeCredits$ | 64                                       | Total credits in a voting committee                            |
| $Supermajority$    | $CommitteeCredits \times \frac{2}{3}$    | Supermajority quorum (43 credits)                              |
| $Majority$         | $CommitteeCredits \times \frac{1}{2} +1$ | Majority quorum (33 credits)                                   |
| $MaxIterations$    | 255                                      | Maximum number of iterations in a single round                 |
| $EmergencyMode$    | $MaxIterations - 10$                     | Iteration at which [Emergency Mode][em] starts                 |
| $DuskKey$          | $pk_\mathcal{Dusk}$                      | A Dusk-owned public key used to sign the [Emergency Block][eb] |


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
| $Round$                           | Integer                 | Current round number              |
| $Iteration$                       | Integer                 | Current iteration number          |
| $\mathsf{B}^c$                    | [`Block`][b]            | Candidate block                   |
| $\mathsf{B}^w$                    | [`Block`][b]            | Winning block                     |
| $\boldsymbol{FailedAttestations}$ | [`Attestation`][att][ ] | Attestations of failed iterations |


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
2. Adjust step base timeouts ([*SetRoundTimeouts*][srt])
3. Start $\mathsf{Quorum}$ message handler ([*HandleQuorum*][hq])
4. For $Iteration$ from 0 to $MaxIterations$
   1. If in Emergency Mode:
      1. Start [*SAIteration*][sai] as a thread
      2. Wait $MaxStepTimeout$ for each step
   2. If not in Emergency Mode:
      1. Execute SA iteration ([*SAIteration*][sai])
   3. If a winning block $\mathsf{B}^w$ has been produced 
      1. Broadcast $\mathsf{B}^w$
      2. Accept $\mathsf{B}^w$ into the chain
      3. End round
5. If we reached $MaxIterations$ without a winning block
   1. If we are a Dusk-owned node
      1. Produce an Emergency Block
   2. Otherwise, stop the SA loop (and wait for some block to be received)

**Procedure**

$\textit{SARound}():$
1. $\texttt{set }$:
   - $\mathsf{B}^c, \mathsf{B}^w = NIL$
2. [*SetRoundTimeouts*][srt]$()$
3. $\texttt{start}($[*HandleQuorum*][hq]$(Round))$
4. $\texttt{for } Iteration = 0 \dots MaxIterations-1 :$
   1. $\texttt{if } (I \ge EmergencyMode):$
      1. $\texttt{start}($[*SAIteration*][sai]$(Round, Iteration))$
      2. $\texttt{wait} (3 \times MaxStepTimeout)$
   2. $\texttt{else}:$
      1. [*SAIteration*][sai]$(Round, Iteration)$
   3. $\texttt{if }(\mathsf{B}^w \ne NIL):$
      1. [*Broadcast*][mx]$(\mathsf{B}^w)$
      2. [*AcceptBlock*][ab]$(\mathsf{B}^w)$
      3. $\texttt{break}$
5. $\texttt{if } (\mathsf{B}^w = NIL)$
   1. If $\mathcal{N} = DuskKey$
      1. [*BroadcastEmergencyBlock*][beb]$()$
   2. $\texttt{stop}($[*SALoop*][sal]$)$

<p><br></p>

#### *SAIteration*
This procedure executes the sequence of [*Proposal*][prop], [*Validation*][val], and [*Ratification*][rat] steps.
The *Proposal* outputs the candidate block $\mathsf{B}^c$ for the iteration; this is passed to *Validation*, which, if a quorum is reached, outputs the aggregated Validation votes $\mathsf{SV}^V$; these are passed to *Ratification*, which, if a quorum is reached, outputs the aggregated Ratification votes $\mathsf{SV}^R$.

If a quorum was reached in both Validation and Ratification, a `Quorum` message is broadcast with the [`Attestation`][atts] of the iteration (i.e. the two `StepVotes` $\mathsf{SV}^V$ and $\mathsf{SV}^R$).

**Algorithm**
1. Run *Proposal* to generate the *candidate* block $\mathsf{B}^c$
2. Run *Validation* on $\mathsf{B}^c$
3. Run *Ratification* on the Validation result
4. If Ratification reached a quorum on $v$: 
   1. Create an attestation $\mathsf{A}$ with the Validation and Ratification votes
   2. Set vote to $(v, \eta_{\mathsf{B}^c})$
      1. Create $\mathsf{Quorum}$ message $\mathsf{M^Q}$
   3. Broadcast $\mathsf{M^Q}$
   4. If the Ratification result is $Success$:
      1. Make $\mathsf{B}^c$ the winning block [*MakeWinning*][mw]
   5. If the Ratification result is $Fail$
      1. Add $\mathsf{A}$ to the $\boldsymbol{FailedAttestations}$ list

**Procedure**
$\textit{SAIteration}(R, I):$
1. $\mathsf{B}^c =$ [*ProposalStep*][props]$(R, I)$
2. $\mathsf{SR}^V =$ [*ValidationStep*][vs]$(R, I, \mathsf{B}^c)$
3. $\mathsf{SR}^R =$ [*RatificationStep*][rs]$(R, I, \mathsf{SR}^V)$
- $\texttt{set}:$
  - $`\_, \_, \mathsf{SV}^V \leftarrow \mathsf{SR}^V`$
  - $v, \eta_{\mathsf{B}^c}, \mathsf{SV}^R \leftarrow \mathsf{SR}^R$
4. $\texttt{if } (v \ne NoQuorum):$
   1. $\mathsf{A} = ({\mathsf{SV}^V, \mathsf{SV}^R})$
   2. $\mathsf{VI} = (v, \eta_{\mathsf{B}^c})$
   3. $\mathsf{M^Q} =$ [*Msg*][msg]$(\mathsf{Quorum}, \mathsf{VI}, \mathsf{A})$
      | Field           | Value                 |
      |-----------------|-----------------------|
      | $PrevHash$      | $\eta_{Tip}$          |
      | $Round$         | $R$                   |
      | $Iteration$     | $I$                   |
      | $Vote$          | $v$                   |
      | $CandidateHash$ | $\eta_{\mathsf{B}^c}$ |
      | $Attestation$   | $\mathsf{A}$          |
   4. [*Broadcast*][mx]$(\mathsf{M^Q})$
   5. $\texttt{if } (v = Success):$
      1. [*MakeWinning*][mw]$(\mathsf{B}^c, \mathsf{A})$
   6. $\texttt{else}:$
      1. $\boldsymbol{FailedAttestations}[I] = {\mathsf{A}}$

<p><br></p>

## Step Timeouts
Each step of the [SA iteration][sai] has a limited timeout, within which the step can succeed. If the timeout expires, the step fails.

Step timeouts serve various purposes. First of all, they are used to keep provisioners aligned with each other: to reach consensus on a candidate block, provisioners need to run the same round and iteration at approximately the same time. Timeouts force the consensus loop to run on a fast pace, without indefinite waits for steps to end. For instance, if the provisioner node gets isolated from the network, it won't get stuck at the same round, but will move on until either an iteration succeeds or a full block is received.

Timeouts are also functional to avoid forks. Without timeouts, a step could run indefinitely until some result is obtained (e.g., a candidate block is received, or a quorum of votes is reached). Dealing with this would require iterations to run in parallel, leading to blocks that could be replaced at any moment by lower-iteration ones. Instead, by having a limited time on each step, provisioners always act in a finite time, allowing iterations to move on. For instance, if the candidate block does not arrive in time, the provisioner votes $NoCandidate$ in the Validation step. Similarly, if the Validation step does not reach any quorum within the timeout, the provisioner votes $NoQuorum$. Moreover, once a vote is cast, no other vote is allowed from the same provisioner (this behavior will be [slashed][sla] in the future). These timeout-induced votes enable blocks that are not produced at the first iteration to be [finalized][fin]. This is because a quorum of negative votes can be used to produce a [Failed Attestation][atts] that proves the iteration failed and can't reach any valid quorum.

Another effect of timeouts is enabling [slashing][sla] a provisioner for not producing a block. While timeouts cannot ensure a block was not produced at all (which is impossible, since the generator could create the block and then postpone its broadcast indefinitely) they allow provisioners to agree on the fact that a majority did not receive such a block.

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

| Name                 | Value  | Description                                            |
|----------------------|--------|--------------------------------------------------------|
| $TimeoutIncrease$    | 2      | Increase amount in case of timeout (seconds)           |
| $MinStepTimeout$     | 2      | Minimum timeout for a single step (seconds)            |
| $MaxStepTimeout$     | 30     | Maximum timeout for a single step (seconds)            |
| $MaxElapsedTimes$    | 5      | Maximum number of elapsed time values stored (seconds) |

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

## Emergency Mode
In extreme cases where most provisioners are offline or isolated, multiple consecutive iterations may fail due to the lack of block generators or voters. In such a situation, the maximum number of iterations for a round may be reached. To avoid ending a round without an accepted block, the SA protocol implements an *emergency mode*.

This mode activates when the iteration number approaches its limit ($MaxIterations$) and consists in disabling the step timeouts. By doing so, the last iterations will be considered active until they reach a quorum on a candidate block ($Valid$ or $Invalid$). In other words, in emergency mode, iterations only end if a candidate block is generated and a quorum is reached in both Validation and Ratification. As a consequence, $NoCandidate$ and $NoQuorum$ votes are disabled in emergency mode.

Iterations in emergency mode are run in parallel (although a sum of the step timeouts is waited before starting a new iteration), with each iteration moving forward only if the current step succeeds. As a consequence, the possibility of forks is increased. In fact, each of the running iterations could reach agreement on a candidate block, possibly generating multiple winning blocks for the same round. Nonetheless, forks are still automatically resolved by always choosing the lowest-iteration block (see [Chain Management][cm]).
<!-- TODO: check the following is implemented: -->
In this respect, note that, if a block is accepted for the round, all active iterations are interrupted. This decreases the chances of other candidate blocks reaching quorum.

### Emergency Block
If even emergency-mode iterations fail to produce a new block, the last iteration is reserved for Dusk-owned nodes to produce an *Emergency Block*.
This block is empty (i.e., it contains no transactions) and signed with a private key belonging to Dusk. The corresponding public key is a [global parameter][cenv] ($DuskKey$) known to all nodes.

If a node receives an Emergency Block it accepts it into its local chain and moves to the next round. Nevertheless, the Emergency Block is $Accepted$ by definition, so it can be replaced if a lower-iteration block is received for the same round (unless it gets finalized due to [Rolling Finality][rf]).

Note that multiple nodes owned by Dusk can produce the exact same block, since it contains no transactions and will thus have the same hash. Thus, no fork can be produced by multiple Emergency Blocks for the same round.

### Procedures

#### *BroadcastEmergencyBlock*
This procedure creates a block $\mathsf{B}$ with no transactions and signed with the private $DuskKey$.

#### *isEmergencyBlock*
This procedure outputs $true$ if the input block $\mathsf{B}$ is a valid Emergency Block.

$`\textit{isEmergencyBlock}(\mathsf{B})`$
- $\texttt{if} (\mathsf{B}.Iteration = MaxIterations)$
- $\texttt{and} (\mathsf{B}.Transactions = NIL)$
- $\texttt{and} (\mathsf{B}.Generator = DuskKey):$
  - $\texttt{output} true$


<!----------------------- FOOTNOTES ----------------------->

[^1]: For the sake of simplicity, we assume the node handles a single provisioner identity.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md -->
[sa]:   #overview

[em]:   #emergency-mode
[eb]:   #emergency-block
[beb]:  #broadcastemergencyblock
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
[lc]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#local-chain
[chb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#chainblock-structure
[sys]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#system-state
[vms]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#vmstate-structure
[fin]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#finality
[rf]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#rolling-finality

[pro]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#provisioners-and-stakes
[sla]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#slashing

[gq]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetQuorum
[gsn]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#GetStepNum
[atts]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestations
[att]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestation
[sc]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#subcommittee
[sb]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#setbit

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
[msg]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#msg
[mx]:    https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#procedures
[qmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#quorum
[rmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#ratification
[cmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#candidate
[bmsg]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/messages.md#block

<!-- Network -->
[net]:   https://github.com/dusk-network/dusk-protocol/tree/main/network/README.md
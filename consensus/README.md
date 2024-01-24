<!-- TODO: mention ProcessBlock -->
<!-- TODO: All nodes can contribute to consensus by collecting votes and generating an Agreement message as soon as a quorum is reached on both Reductions -->
<!-- TODO: Define sth like Candidate "signature": block_hash, round, iteration -->
# Succinct Attestation
**Succinct Attestation** (**SA**) is a permissionless, committee-based Proof-of-Stake consensus protocol. 
The protocol is run by Dusk stakers, known as ***provisioners***, which are responsible for generating and validating new blocks.

Provisioners participate in turns to the production and validation of each new block of the ledger. Participation is determined by the [*Deterministic Sortition*][ds] (*DS* in short) algorithm, which is used to extract a unique *block generator* and unique *voting committees* among provisioners, in a decentralized, non-interactive way.

Before continue reading, please make sure you are familiar with the system [Basics][bas] and the document [Notation][not].

### ToC
- [Overview](#overview)
- [SA Environment](#environment)
- [SA Algorithm](#sa-algorithm)
  - [SAConsensus](#saconsensus)
  - [SALoop](#saloop)
  - [SARound](#saround)
  - [SAIteration](#saiteration)
  - [IncreaseTimeout](#increasetimeout)


## Overview
The SA protocol proceeds in ***rounds***, with each round adding a new block to the chain.
In turn, each round proceeds in ***iterations***, with each iteration aiming at generating a candidate block and reaching agreement among provisioners.

Each iteration is composed of three phases, or ***steps***:
  1. ***Proposal***: in this step, a *generator*, extracted via [*DS*][ds], creates a new candidate block and broadcasts it to the network using a $\mathsf{Candidate}$ message;
  
  2. ***Validation***: in this step, a committee of provisioners, extracted via [*DS*][ds], votes on the validity of the candidate block via $\mathsf{Validation}$ messages. The committee can reach a successful *quorum* with a supermajority ($\frac{2}{3}$) of $Valid$ votes or an unsuccessful quorum with a majority ($\frac{1}{2}+1$) of $Invalid$ votes, cast for an invalid candidate, or $NoCandidate$ votes, cast when the candidate is unknown.
  
  3. ***Ratification***: in this step, a new committee of provisioners, extracted via [*DS*][ds], votes on the result of the previous Validation step via $\mathsf{Ratification}$ messages. If a quorum was reached they cast the winning Validation vote, otherwise they will vote $NoQuorum$. If a quorum is reached, a $\mathsf{Quorum}$ message is broadcast, containing a $\mathsf{Certificate}$ with the votes of the two steps.
  Similar to Validation, the step succeeds if a supermajority quorum of $Valid$ votes is received, and it fails with a majority quorum of $Invalid$, $NoCandidate$, or $NoQuorum$ votes.
  If no quorum is reached, the step result is unknown.

If a quorum of $Valid$ votes is reached in both the Validation and Ratification steps, the candidate block generated in the Proposal step is accepted as the new tip of the chain.
Conversely, if the Ratification result is a failure or unknown, a new iteration is executed (that is a new candidate block is produced and voted upon).

The round terminates when an iteration is successful.

A maximum number of 255 iterations is executed within a single round.

## Environment
We here define global parameters and state variables of the SA protocol, used and shared by all consensus procedures.

The SA environment is composed of:
- *global parameters*: network-wide parameters used by all nodes of the network using a particular protocol version;
- *chain state*: represent the current system state, as per result of the execution of all transactions in the blockchain;
- *round state*: local variables used to handle the consensus state 

Additionally, we denote the node running the protocol with $\mathcal{N}$ and refer to its provisioner[^1] keys as $sk_\mathcal{N}$ and $pk_\mathcal{N}$.

**Global Parameters**
All global values (except for the genesis block) refer to version $0$ of the protocol.

| Name                    | Value          | Description                                          |
|-------------------------|----------------|------------------------------------------------------|
| $GenesisBlock$          | $\mathsf{B_0}$ | Genesis block of the network                         |
| $Version$               | 0              | Protocol version number                              |
| $Dusk$                  | 1.000.000.000  | Value of one unit of Dusk (in lux)                   |
| $MinStake$              | 1.000          | Minimum amount of a single stake (in Dusk)           |
| $Epoch$                 | 2160           | Epoch duration in number of blocks                   |
| $CommitteeCredits$      | 64             | Total credits in a voting committee                  |
| $Quorum$                | $CommitteeCredits \times \frac{2}{3}$ (43) | Quorum threshold         |
| $NegativeQuorum$        | $CommitteeCredits-Quorum+1$ (22) | Quorum threshold for NIL votes     |
| $MaxIterations$         | 255            | Maximum number of iterations for a single round      |
| $RollingFinalityBlocks$ | 5              | Number of Attested blocks for [Rolling Finality][rf] |
| $InitTimeout$           | 5              | Initial step timeout (in seconds)                    |
| $MaxTimeout$            | 60             | Maximum timeout for a single step (in seconds)       |
| $BlockGas$              | 5.000.000.000  | Gas limit for a single block                         |


**Chain State**
<!-- TODO: Add Type column -->
| Name                 | Description                             |
|----------------------|-----------------------------------------|
| $Tip$                | Current chain tip (last block)          |
| $State$              | Current system state                    |
| $Provisioners$       | Current set of (eligible) provisioners  |

**Round State**
| Name                          | Description                          |
|-------------------------------|--------------------------------------|
| $Round_{SA}$                  | Round number                         |
| $Iteration_{SA}$              | Iteration number                     |
| $\mathsf{B}^c$                | Candidate block                      |
| $\mathsf{B}^w$                | Winning block                        |
| $\tau_{Proposal}$             | Current timeout for Proposal      |
| $\tau_{Validation}$           | Current timeout for First Reduction  |
| $\tau_{Ratification}$         | Current timeout for Second Reduction |
| $\boldsymbol{PrevIterations}$ | Certificates of failed iterations    |
<!-- | $\mathsf{V}^1$                | StepVotes of First Reduction         |
| $\mathsf{V}^2$                | StepVotes of Second Reduction        | -->


## Procedures
The SA consensus is defined by the [*SAConsensus*][sac] procedure, which executes an infinite loop ([*SALoop*][sal]) of rounds ([*SARound*][sar]), each executing one or more iterations ([*SAIteration*][sai]) until a *winning block* ($\mathsf{B}^w$) is produced for the round, becoming the new $Tip$ of the chain ([*AcceptBlock*][ab]).
The consensus loop could be interrupted when receiving a valid $\mathsf{Block}$ ([*ProcessBlock*][pb]) or $\mathsf{Quorum}$ ([*ProcessQuorum*][pq]) message, which could make a block accepted as the new $Tip$.

<!-- DOING -->

### *SAConsensus*
The *SAConsensus* procedure is the entry point of a consensus node. Upon booting, the node checks if there is a local state saved and, if so, loads it. Otherwise, it starts from the *Genesis Block*. Then, it probes the network to check if it is in sync or not with the blockchain. If not, it starts a synchronization procedure. When in sync, it executes the consensus algorithm per rounds. At the end of each round, if a winning block has been produced, it updates the state and starts a new round. 

If, at any time, the node falls behind the network, the synchronization procedure is run before restarting the rounds loop.

***Algorithm***

1. Load local state
2. If there is no saved state
   1. Set $Tip$ to Genesis Block
3. Otherwise
   1. Check state validity
   2. Set $State$ to loaded state
   3. Set $Tip$ to last block
4. Start SA loop procedure ([*SALoop*][sal])

***Procedure***

$\textit{SAConsensus}():$
1. $S =$ *LoadState*$()$
2. $\texttt{if } (S = NIL):$
   1. $Tip = GenesisBlock$
3. $\texttt{else}:$
   1. *ValidateState*$(S)$    <!-- TODO -->
   2. $State = S.State$
   3. $Tip = S.Tip$
4. $\texttt{start}$([*SALoop*][sal])
<!-- TODO: MessageHandler loop to handle Block and Quorum messages -->

### *SALoop*
The *SALoop* procedure executes SA rounds (*SARound*). We define this as a separate procedure to allow it to be stopped and restarted when synchronizing the blockchain.

The procedure executes an infinite loop of SA rounds. At each iteration, the $Round_{SA}$ global variable is set to $Tip$'s height plus one; then the SA round is executed; if the round produced a winning block, it executes the state transition using this block. If any round ends with no winning block, the consensus process is considered as faulty and halts. Such an event requires a manual recovery procedure.

***Algorithm***

1. Loop:
   1. Set $Round_{SA}$ to $Tip$'s height plus one
   2. Execute Round $Round_{SA}$ to produce winning block $\mathsf{B}^w$
   3. If no winning block has been produced, halt consensus
   4. Execute state transition

***Procedure***

$\textit{SALoop}():$
1. $\texttt{loop}:$
   1. $Round_{SA} = Tip.Height + 1$
   2. $\mathsf{B}^w =$ [*SARound*][sar]$(Round_{SA})$
   3. $\texttt{if } (\mathsf{B}^w = NIL):$ 
       - $\texttt{stop}$
   4. $State =$ [*AcceptBlock*][ab]$(\mathsf{B}^w)$

### SARound
The *SARound* procedure handles the execution of a consensus round: first, it initializes the *Round State* variables; then, it starts the Ratification process in background, and starts executing SA iterations. If, at any time, a winning block is produced by the Ratification process, the round stops.

***Algorithm***

1. Set variables:
   - Initialize Proposal and Reduction timeouts
   - Set candidate and winning block to $NIL$
   - Set iteration to 0
2. Start Ratification process
3. While iteration number is less than $MaxIterations$ and no winning block has been produced
   1. Execute SA iteration
4. If we reached $MaxIterations$ without a winning block
   1. Output $NIL$
5. Otherwise, broadcast the winning block <!-- TODO: move this to SAConsensus ? -->
6. Output the block

***Procedure***

$\textit{SARound}(Round_{SA}):$
1. $\texttt{set }$:
   - $\tau_{Proposal}, \tau_{Reduction_1}, \tau_{Reduction_2} = InitTimeout$
   - $\mathsf{B}^c, \mathsf{B}^w = NIL$
   - $Iteration_{SA} = 0$
2. $\texttt{start}$([*Ratification*][rata]$(Round_{SA}))$
3. $\texttt{while } (\mathsf{B}^w = NIL) \texttt{ and } (Iteration_{SA} \le MaxIterations)$
   1. [*SAIteration*][sai]$(Round_{SA}, Iteration_{SA})$
4. $\texttt{if } (\mathsf{B}^w = NIL)$
   1. $\texttt{output } NIL$
5. [*Broadcast*][mx]$(\mathsf{B}^w)$
6. $\texttt{output } \mathsf{B}^w$


### SAIteration
This procedure executes a sequence of *Proposal*, to generate a new candidate block ($\mathsf{B}^c$) for the current round and iteration, and two *Reduction* steps, to vote on the candidate block (if any). Quorum votes of first and second Reduction (if any) are stored in $\mathsf{V}^1$ and $\mathsf{V}^2$, respectively. If the votes of both committees reach a quorum, an [Agreement][amsg] message broadcasted, containing all quorum votes.


***Algorithm***
1. Run Proposal to generate *candidate* block $\mathsf{B}^c$
2. Run first Reduction on $\mathsf{B}^c$
3. Run second Reduction on $\mathsf{B}^c$
4. If any of the two Reductions failed
   1. Create a certificate with the aggregated votes
   2. Add the failed-iteration certificate to $\boldsymbol{PrevIterations}$
5. and this node $\mathcal{N}$ is in the second Reduction committee:
   1. Create $\mathsf{Agreement}$ message $\mathsf{M}^A$ with both Reduction votes
   2. Broadcast message $\mathsf{M}^A$

***Procedure***

$\textit{SAIteration}(Round, Iteration):$
- $r2Step = Iteration\times 3 + 2$
- $C^{R2} =$ [*DS*][dsa]$(Round,r2Step,CommitteeCredits)$
1. $\mathsf{B}^c =$ [*Proposal*][ps]$(Round, Iteration)$
2. $(v^1, \mathsf{V}^1) =$ [*Reduction*][reda]$(Round, Iteration, 1, \mathsf{B}^c)$
3. $(v^2, \mathsf{V}^2) =$ [*Reduction*][reda]$(Round, Iteration, 2, \mathsf{B}^c)$
4. $\texttt{if } (v^1 = NIL) \texttt{ or } (v^1 = NIL)$
   1. $\mathsf{C} = {\mathsf{V}^1, \mathsf{V}^2}$
   2. $\boldsymbol{PrevIterations}[Iteration] = {\mathsf{C}}$
5. $\texttt{else if} (pk_\mathcal{N} \in C^{R2}):$
    1. $\mathsf{M}^A =$ [*Msg*][msg]$(\mathsf{Agreement}, [\mathsf{V}^1,\mathsf{V}^2])$
        | Field       | Value                         | 
        |-------------|-------------------------------|
        | $Header$    | $\mathsf{H}_\mathsf{M}$       |
        | $Signature$ | $\sigma_\mathsf{M}$           |
        | $RVotes$    | $[\mathsf{V}^1,\mathsf{V}^2]$ |

    2. [*Broadcast*][mx]$(\mathsf{M}^A)$


### IncreaseTimeout
*IncreaseTimeout* doubles a step timeout up to $MaxTimeout$.

***Parameters***
- $\tau_{Step}$: $Step$ timeout to increase (where $Step$ can be $Proposal$, $Reduction1$, or $Reduction2$)

***Procedure***

$\textit{IncreaseTimeout}(\tau_{Step}):$
- $\tau_{Step} =$ *Max*$(\tau_{Step} \times 2, MaxTimeout)$

<!-- TODO: Define $GetStepNumber$ 
Proposal: (Block.Iteration) \times 3 + 1
red1: (Block.Iteration) \times 3 + 1
red2: (Block.Iteration) \times 3 + 2
-->

<!----------------------- FOOTNOTES ----------------------->

[^1]: For the sake of simplicity, we assume the node handles a single provisioner identity.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md -->
[sa]:   #overview
[env]:  #environment
[sac]:  #saconsensus
[sar]:  #saround
[sai]:  #saiteration
[sal]:  #saloop
[it]:   #increasetimeout

[net]: https://github.com/dusk-network/dusk-protocol/tree/main/network
[not]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/notation
[bas]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics

<!-- Chain Management -->
[ab]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#acceptblock
[pb]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#processblock
[rf]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/chain-management/README.md#rolling-finality
<!-- Sortition -->
[ds]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/
[dsa]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#algorithm
[sc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#subcommittee
[sb]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#setbit
<!-- Proposal -->
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal/
[ps]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal/README.md#proposalstep
<!-- Reduction -->
[red]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/
[reda]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/reduction/README.md#reduction-algorithm
<!-- Ratification -->
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/
[rata]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md#ratification-algorithm
<!-- Messages -->
[msg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-creation
[mx]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-exchange
[mh]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#message-header
[amsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#agreement-message
[rmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#ratification-message
[cmsg]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/messages/README.md#candidate-message
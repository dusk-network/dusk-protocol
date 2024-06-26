# Staking
This section describes staking in Dusk. In particular, it formally defines stakers, known as *provisioners*, and describes the incentive system aimed at fostering participation and discouraging misbehaving.

**ToC**
  - [Provisioners and Stakes](#provisioners-and-stakes)
    - [Epochs and Eligibility](#epochs-and-eligibility)
  - [Incentives](#incentives)
    - [Rewards](#rewards)
    - [Slashing](#slashing)

<p><br></p>

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

<p><br></p>

## Incentives
Given the nature of the SA consensus protocol, it is of paramount importance that provisioners participate when selected by the [Deterministic Sortition][ds] algorithm. In fact, offline provisioners can slow down the network, and, in extreme cases, even stop block production. Specifically, the provisioner selected for the [Proposal][prop] step must be online to produce the candidate block; similarly, a supermajority of voters is required to be online when the [Validation][val] or [Ratification][rat] steps occur, to reach a quorum on the candidate block. If any of the two conditions are not met, the iteration fails and a new one is needed to create a new block.
To minimize the risk of failure, provisioners are incentivized (or, equally, disincentivized) by means of *rewards* and *penalties*.

The information required to apply rewards and penalties is always included in a block. In particular, we use [attestations][atts] as a single source of truth, with the addition of *proofs of faults* to demonstrate some misbehaviors. All incentives are handled by the [VM][vm] when performing the *state transition* of the accepted block.

### Rewards 
The most basic reward is, for obvious reasons, the block reward, which is given to the generator of a candidate block that gets accepted into the chain.
This reward consists of two parts: the assignment of newly-emitted coins, and a portion of the fees of the transactions included in the block (for more details on fee distribution see the [Economic Protocol][ep]). In particular, the block generator is assigned 90% of the block emission (see the Dusk [emission schedule][tok]).

On the other hand, a small reward is also assigned to the voters of an accepted block[^2]. This not only incentivizes provisioners to stay online, but also disincentivizes skipping the vote at lower iterations. In fact, since the block producers of all iterations in the current round are known, if a block producer of a later iteration is in the voting committee of the current iteration, they might be incentivized to not vote in order to get to produce their block.
Additionally, we incentivize the block producer to include as many voters as possible by making a portion of the block reward proportional to the number of signatures in the block attestation. This is intended to disincentivize the block generator from cherry-picking signatures to penalize other provisioners.

Rewards are not added to the provisioner stake, but they are collected into a $Reward$ amount, from which the provisioner can later withdraw coins. This is done to limit the power-increasing effect of provisioners with bigger stakes being selected more (see [Deterministic Sortition][ds]).

### Slashing
The following behaviors are subject to slashing:
- Missed block: if, in the [Proposal][prop] step, the selected generator fails to broadcast the candidate block; the slash takes effect when accepting a block with a [Failed Attestation][atts] of $NoCandidate$ quorum;
- Invalid block: if a candidate block is deemed invalid by the [Validation][val] step; the slash takes effect when accepting a block with a Failed Attestation of $Invalid$ quorum;
- Double voting: if a provisioner signs for two different votes regarding the same candidate block; the slash takes effect when an ad-hoc transaction is included in a block, containing the two conflicting signatures[^2];
- Double block: if a generator broadcasts two candidates for the same iteration; the slash takes effect when an ad-hoc transaction is included in a block, containing the two conflicting signatures.

Currently, the slash amount is equal to the block reward (which varies according to the [emission schedule][tok]).

Slashing is applied to the provisioner's reward first and then to the actual stake. That is, if the provisioner previously earned some rewards, then slashing affects this amount first. If the reward amount reaches zero, then slashing is applied to the actual stake.

Note that slashing has an immediate effect, in contrast with [staking][pro], which takes effect at the beginning of the second epoch after the stake operation.

<!----------------------- FOOTNOTES ----------------------->

[^1]: Note that an epoch refers to a specific set of blocks and not just to a number of blocks; that is, an epoch starts and ends at specific block heights.

[^2]: Note that rewards related to quorum voters are not yet implemented, so no specific reward amount has been yet decided. The same applies to slashing for double vote and double candidate.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md -->

[pro]: #provisioners-and-stakes
[epo]: #epochs-and-eligibility
[inc]: #incentives
[rew]: #rewards
[sla]: #slashing

<!-- Basics -->
[atts]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestations

<!-- Protocol -->
[cenv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#environment
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md
[ds]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md
[dsp]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md#deterministic-sortition-ds

<!-- Tokenomics -->
[ep]:      https://github.com/dusk-network/dusk-protocol/tree/main/economic-protocol
[tok]:     https://docs.dusk.network/learn/economy/tokenomics/#token-emission-schedule

<!-- TODO: these links point to missing pages -->
[vm]:      https://github.com/dusk-network/dusk-protocol/tree/main/vm
[c-stake]: https://github.com/dusk-network/dusk-protocol/tree/main/contracts/stake


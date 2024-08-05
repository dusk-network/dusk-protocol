# Staking
This section describes staking in Dusk. In particular, it formally defines stakers, known as *provisioners*, and describes the incentive system aimed at fostering participation and discouraging misbehaving.

**ToC**
<!-- TODO: Add "Roles" section, describing Block Generator and Voters -->

  - [Provisioners and Stakes](#provisioners-and-stakes)
    - [Epochs and Eligibility](#epochs-and-eligibility)
  - [Incentives](#incentives)
    - [Future-Generator Incentive Problem](#future-generator-incentive-problem)
    - [Rewards](#rewards)
      - [Generator Reward](#generator-reward)
      - [Voters Reward](#voters-reward)
    - [Penalties](#penalties)
      - [Faults](#faults)
      - [Suspension](#suspension)
      - [Soft Slashing](#soft-slashing)
      - [Hard Slashing](#hard-slashing)

<p><br></p>

## Provisioners and Stakes
A Provisioner is a user that locks a certain amount of its own Dusks as *stake* (see [*Stake Contract*][c-stake]). Any user can do so by broadcasting a [`stake` transaction][stx].
Formally, we define a ***Stake*** as

$$\mathsf{S} = (Amount, Height),$$

where $Amount$ is the quantity of locked Dusks, and $Height$ is the height of the block where the `stake` transaction was included. The minimum value for a stake is defined by the [global parameter][cenv] $MinStake$, and is currently equivalent to 1000 Dusk.


In turn, we define a ***Provisioner*** $\mathcal{P}$ as:

$$\mathcal{P}=(pk_\mathcal{P}, \mathsf{S}_\mathcal{P}),$$

where $pk_\mathcal{P}$ is the BLS public key of the provisioner and $\mathsf{S}_\mathcal{P}$ is the stake belonging to $\mathcal{P}$.

The stake amount directly influences the probability of being extracted in the [Deterministic Sortition][ds] algorithm: the higher the stake, the more the provisioner will be selected, on average.

Whenever a Provisioner wants to unlock its Stake, it can do so by broadcasting an `unstake` transaction.

At any time, a unique set of provisioners can be derived from the set of previous `stake` and `unstake` operations. We refer to such set as the ***Provisioner Set***

### Epochs and Eligibility
Being in the Provisioner Set is a necessary condition for participating in the consensus protocol. However, it is not sufficient to be selected by the [Deterministic Sortition][ds] algorithm.

In particular, only eligible stakes are able to participate. Formally, we say a Stake $\mathsf{S}$ is 
***eligible*** in a [round][sa] $R$ if it satisfies the following conditions:
  1. $Amount \ge MinStake$
  2. $R \gt Height_\mathsf{S} + M$

where $M$ is the *maturity* period and is defined as:

$$M = 2{\times}Epoch - (Height_\mathsf{S} \mod Epoch),$$

where $Epoch$ corresponds to a fixed number of blocks defined as [global parameter][cenv] (currently set to $2160$).

Note that we use the term ***Epoch*** to refer to a specific range of $Epoch$ blocks, starting from height $0$. For instance, the 1st Epoch corresponds to blocks $0$ to $Epoch-1$, the 2nd Epoch starts at block $Epoch{\times}2$, and so on. As such, "time" in the Dusk blockchain is effectively divided into Epochs, which mark participation in the consensus protocol. 

For instance, the Maturity period of a Stake, after which it becomes eligible, corresponds to the remainder of the Epoch in which the `stake` transaction was included, plus another full Epoch. As a consequence, all stakes become eligible at the beginning of a new Epoch.

The concept of epochs and eligibility is particularly useful for the *pre-verification* of blocks that are at a greater height than a node's [Tip][lc]. In fact, nodes can predict, to some extent, the Provisioner Set of the Tip's current epoch and that of the following one, thus being able to discard blocks and votes from Provisioners that are not in such sets.

<p><br></p>

## Incentives
Given the nature of the SA consensus protocol, it is of paramount importance that provisioners participate when selected by the [Deterministic Sortition][ds] algorithm. In fact, offline provisioners can slow down the network, and, in extreme cases, even stop block production. Specifically, the provisioner selected for the [Proposal][prop] step must be online to produce the candidate block; similarly, a supermajority of voters is required to be online for the [Validation][val] or [Ratification][rat] steps to reach a quorum on the candidate block. If any of the two conditions are not met, the iteration fails and a new one is needed to create a new block.
To minimize the risk of failure, provisioners are incentivized by means of *rewards* and disincentivized through *penalties*.

The information required to apply rewards and penalties is always included in a block. In particular, we use [attestations][atts] as a single source of truth, with the addition of [*proofs of faults*][fau] to demonstrate some misbehaviors. All incentives are handled by the [VM][vm] when performing the *state transition* of the accepted block.

### Future-Generator Incentive Problem
Due to the multiple-iteration design of the SA consensus protocol, there is an intrinsic incentive for generators of future iterations not to vote in previous iterations (if selected as voters).
In other words, when a provisioner is selected both as voter in an iteration $I$ and as a generator in an iteration $J > I$[^1], it is more convenient for it to make iteration $I$ fail and reach iteration $J$ where it could potentially earn the block reward. We refer to this as the *Future-Generator Incentive Problem*.

To mitigate this harmful incentive, we include several mechanisms: 
- voter rewards: provisioners gain a small reward for voting for a candidate block; this implies the future-generator has something to lose by not voting when selected. The choice is between an (almost) certain reward now or a possible bigger reward later (conditional to all previous iterations to fail);
- ExtraCredits reward to generators: to ensure all provisioners are equally incentivized we want as many of them as possible to be included the block certificate (which is used to assign reward), we make part of the generator reward conditional on the inclusion of all known votes;
- exclude next-iteration generator from current-iteration voters: given that the probability for a future-iteration generator to gain a block reward is inversely proportional to the iteration number in which they are selected as a generator, the provisioner with the highest such probability at iteration $I$ is the generator of iteration $I+1$; to avoid "temptation" we then exclude such generator from the set of voters;
- limit the number of iterations: as the number of future-iteration generators is directly proportional to the number of iterations, we keep the maximum number of iterations limited to the minimum needed to ensure security.

### Rewards
Rewards are assigned with each new block accepted into the chain. In particular, for each block, a *block reward*, consisting of newly-emitted coins (according to the Dusk [emission schedule][tok]) and all the transaction fees, is distributed among Dusk and the provisioners that contributed to the block production: the block generator and the voters of the Validation and Ratification committees.
In particular, as Dusk is assigned 10% of the block reward, the remainder is divided into two portions: the 90% goes the generator (we call this amount the *generator reward*), and the 10% goes to the block voters (we call this amount the *voters reward*).

<!-- (for more details on fee distribution see the [Economic Protocol][ep]).  -->

These rewards are not only a generic incentive for provisioners to stay online, but also aims at mitigating the [Future-Iteration Incentive][fgi] problem. 

In addition, we also incentivize the block generator to include as many votes as possible in the [block certificate][cert] by making a portion of the block reward subject to the number of signatures included. This is intended to disincentivize the block generator from cherry-picking signatures to penalize other provisioners.

Rewards are not added to the provisioner stake, but they are collected into a $Reward$ amount, from which the provisioner can later withdraw coins. This is done to limit the power-increasing effect of provisioners with bigger stakes being selected more (see [Deterministic Sortition][ds]).

#### Generator Reward
The *generator reward* constitutes the 80% of the block reward. This amount is further split into two: a fixed part (70% of the block reward) and variable part (10% of the block reward) subject to the inclusion of extra votes in the block certificate.

In particular, we give a quota of the variable part for each extra credit beyond the quorum threshold. In other words, considering two committees (Validation and Ratification) of 64 credits and a quorum of 43 credits, we split the variable part of the generator reward into 42 quotas ($(64-43) \times 2$). The block generator will receive as many quotas as the number of credits beyond 86 ($43 \times 2$).

Formally:

 $$ GeneratorReward = (BlockReward{\times}0.7) + [Quota_{GR} \times (CertificateCredits-QuorumCredits)],$$

 where: $BlockReward$ is the sum of the block emission and transaction fees, $CertificateCredits$ is the total number of credits in the block certificate, and:
 
 $$ QuorumCredits = Supermajority \times 2 ,$$
 
 $$ ExtraCredits = (CommitteeCredits \times 2) - QuorumCredits ,$$

 $$ Quota_{GR} = \frac{BlockReward{\times}0.1}{ExtraCredits} ,$$

where $CommitteeCredits$ and $Supermajority$ are defined as [global parameters][cenv].


For instance, if the votes in the certificate sum up to 100 credits, the generator will get 70% of the block reward plus 14/42 of the variable part (i.e. 3.33%), for a total reward of 73.33% of the block reward. Including all votes earns the generator the full generator reward (80% of the block reward).

Note that, depending on the amount, a small fraction of $BlockReward$ could be lost when splitting it into quotas. Specifically, $Quota_{GR} \times ExtraCredits$ can be less than $BlockReward{\times}0.1$ (the variable part of the Generator Reward). To take this into account, when $CertificateCredits = QuorumCredits + ExtraCredits$ we consider $GeneratorReward = BlockReward{\times}0.8$ (that is, we assign the totality of the variable part). Otherwise, we consider any extra fraction as burned along with unassigned quotas.


#### Voters Reward
The *voters reward* assign the 10% of the block reward to the provisioners whose vote is included in the certificate. Note that this means this reward is earned by voters of the previous block, instead of the current one. This delay is necessary because the certificate of a block, which establish a definitive set of votes, is only included in the next block.

The reward is distributed in the following way: the amount is split into as many quotas as the number of credits (64 per committee, hence 128 quotas in total); each voter in the certificate gets as many quotas as the credits it has in the committee. 

Formally:

 $$ VoterReward = Quota_{VR} \times VoterPower ,$$

 where: $VoterPower$ is the [*power*][vc] (i.e. the number of credits) of the voter in the committee, and $Quota_{VR}$ is defined as follows:

 $$ Quota_{VR} = \frac{BlockReward{\times}0.1}{CommitteeCredits},$$
 
 where: $BlockReward$ is the sum of the block emission and transaction fees, and $CommitteeCredits$ is defined as [global parameter][cenv].

By giving rewards proportionally to the voters' credits we acknowledge the fact that votes from more powerful members (i.e. with more credits in the committee) are indeed more relevant to reach the quorum, so it makes sense to give them a bigger incentive.
Moreover, voters with more credits are also more likely to be generators in the following iterations; by giving them a bigger incentive, we disincentivize their temptation to skip voting.

Note that, if the certificate contains less credits then the size of both committees, some quotas are not assigned. In particular, unassigned quotas get effectively burned. This strategy entails a fixed amount is given to each voter, regardless of how many voters are in the certificate. In tur, this prevents possible incentives for provisioners to misbehave in order to accrue their share of the reward. The conditional part of the generator reward, which is tied to the inclusion of as many votes as possible, is also closely related to this point.


### Penalties
We currently employ two forms of penalties: *suspension* and *slashing*. Suspension is the exclusion of a misbehaving provisioner from the eligible set for a certain number of blocks. In contrast, slashing is the act of forcibly removing a certain amount from the provisioner stake.
In turn, slashing is divided into *soft slashing*, which simply moves part of the stake to the provisioner $Reward$ amount (which do not count in the [sortition][ds] process), and *hard slashing* which effectively burns part of the stake.

Note that slashing takes effect immediately, altering the provisioner set in the middle of an epoch. This is in contrast with [staking][pro], which takes effect at the beginning of the second epoch after the stake operation.

#### Faults
The application of penalties is based on the type of misbheavior, or *fault*. We currently consider two degree of faults: *minor faults* and *major faults*. Minor faults are punished with suspension and soft slashing; in contrast major faults are punished with hard slashing.

Minor faults are:
- Missed block: if, in the [Proposal][prop] step, the selected generator fails to broadcast the candidate block; the punishment takes effect when accepting a block that includes a [Fail Attestation][atts] of $NoCandidate$ quorum;

Major faults are:
- Invalid block: if a candidate block is deemed invalid by the [Validation][val] committee, and it's confirmed in the [Ratification][rat] step; the punishment takes effect when accepting a block with a Fail Attestation of $Invalid$ quorum;
- Double voting: if a provisioner publish two conflicting votes for the same candidate block; the punishment takes effect when a proof of the misbehavior, consisting in the two headers and the two signatures, is included in a block;
- Double block: if a generator broadcasts two different candidates for the same iteration; the punishment takes effect when a proof of the misbehavior, consisting in the two headers and the two signatures, is included in a block.


#### Suspension
This measure consists in removing a provisioner from the [eligible set][epo] for a number of [*epochs*][epo]. Doing this excludes the provisioner from the [Deterministic Sortition][ds] process, preventing it from being selected as block generator or committee member. This measure primarily aims at excluding offline provisioners from consensus, improving liveness and stability.

Suspension works as follows:
 - each provisioner has a number of *warnings* ($Warnings_\mathcal{P}$) before being penalized; at each fault, $Warnings_\mathcal{P}$ is decreased and, when reaching 0, the fault is penalized with suspension. We currently consider an initial value of $Warnings_\mathcal{P} = 1$ (i.e., each provisioner can commit one fault without incurring in any penalty);
 - when punished, the provisioner is excluded for 1 epoch
 - when becoming eligible again:
   - if the provisioner produces a block or a vote, $Warnings_\mathcal{P}$ is reset to its initial value,
   - if the provisioner commits a new fault, it gets suspended for as many epochs as the consecutive number of faults it committed; in other words, if the provisioner is suspended of the $n\text{th}$ consecutive time, its suspension is of $n$ epochs.

#### Soft Slashing
When a provisioner gets suspended, a part of its stake is moved to its $Rewards$ amount. By doing so, the provisioner does not lose money but its weight in the sortition process is reduced, thus reducing its probability of being selected as generator or voter.

The slashed amount follows this rule: if this is the $n\text{th}$ consecutive fault, soft-slash the $(n \times 10)$\% of the stake (that is, the 10\% the first time, the 20\% the second time, and so on). When the stake goes beyond the minimum stake amount (see $MinStake$ [global parameter][cenv]), the stake gets frozen and can only be recovered by unstaking and restaking again. <!-- This is currently not true, since Reward can be withdrawn any time -->

Generally speaking, bigger stakes are able to keep their eligibility for longer than smaller ones. For instance, on one extreme, a minimum stake of 1000 would get frozen after a single suspension, while a stake of 10.000.000 would get frozen at the 10th suspension. Nevertheless, all stake are ensured to get frozen after 10 consecutive faults (as at the 10th fault, the slash would be 100% of the current stake). In other words, there is an upper bound of 10 consecutive faults for all provisioners, regardless of their stake amount.

#### Hard Slashing
Hard slashing consists in the burning of part of a provisioner's stake.

The logic to determine the slashed amount and the suspension time follows the same approach as soft slashing, with the quantities being multiplied by the number of consecutive faults ($n$). However, we additionally consider different levels of *severity*, which acts as a multiplier of the base slash amount (10%). Specifically, the hard slash amount is:

$$ (severity \times 10) \times n $$

The severity parameter is meant to differentiate between faults that can be committed by mistake from those that we consider as an attack. For instance, block can be voted as invalid if produced with an older version of the protocol or with a buggy client. On the other hand, publishing double blocks or double votes is considered as intentional. 
We currently consider the Invalid Block case has having $severity = 1$ and the Double Block and Double Vote cases has having $severity = 2$. 

Note that there are no *warnings* for hard slashing.

<!----------------------- FOOTNOTES ----------------------->
[^1]: Note that at the beginning of a new round is it possible to calculate all generators and voters for all iterations in the round. This is because the Deterministic Sortition only depends on previous-round data (plus the step for which the extraction is done).

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md -->

[pro]: #provisioners-and-stakes
[epo]: #epochs-and-eligibility
[inc]: #incentives
[fgi]: #future-generator-incentive-problem
[rew]: #rewards
[pen]: #penalties
[fau]: #faults

<!-- Basics -->
[lc]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#local-chain

[vc]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#voting-committees
[atts]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#attestations
[cert]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#block-certificate

<!-- Protocol -->
[sa]:   https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#overview
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
[stx]:     https://github.com/dusk-network/dusk-protocol/tree/main/contracts/stake

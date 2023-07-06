# Deterministic Sortition
*Deterministic Sortition* is the non-interactive process used in the [Succinct Attestation](../README.md) protocol to select the *Block Generator* during the [*Attestation*](../attestation/README.md) phase, and the members of the *Voting Committees* during the [*Reduction*](../reduction) phases.

## Voting Committees
<!-- TODO: use weight instead of influence? -->
A committee is formed by a set of provisioners (called *members*), each having a certain weight, or influence, in the voting process. Influence is expressed as a number of _voting credits_, which are distributed among members according to the Deterministic Sortition process.
Thus, the influence of each member within a committee corresponds to the number of credits he has. When deciding on a block, each vote is multiplied by the number of credits assigned to the caster member. As an example, if a member is assigned 3 credits, his vote will be counted 3 times.

Voting committees have a pre-defined total number of credits, called *Credit Pool*, which determines the degree of distribution among provisioners and, indirectly, the maximum number of committee members. As of today, the Credit Pool size is set to 64.

Formally, given a Chain $\Gamma$, we define a committee for round $r$ and step $s$ as 
$$ C^r_s = Committee(\Gamma, r,s)$$
<!-- \{ (pk, c) : (pk, credits) \in C\} -->


### Block Generator Extraction
Note that the selection of a block generator also follows the Deterministic Sortition procedure.
To this purpose, a voting committee is created with a Credit Pool of 1 unit. The provisioner being assigned such credit becomes the block generator for the round.



## Algorithm
The Deterministic Sortition process creates a committee of Provisioners by randomly assigning a number of voting credits (the *Credit Pool*) to eligible provisioners (see [Eligibility](../README.md#participants)). 

The process takes as inputs the set of eligible provisioners, the pool size (i.e., the number of credits to assign), the *seed* included in the last block, and the current *round* and *step* (see [Consensus Workflow](../README.md#workflow)).
The output of the process is the set of selected provisioners with the number of assigned credits.

Formally:
```
DeterministicSortition (
    Provisioner[]   EligibleProvisioners, 
    int             PoolSize, 
    byte[]          seed, 
    int             round, 
    int             step
) --> Committee
```
where `Committee` is defined as:
```
Committee {
    Member[]
}

Member {
    byte[]      PubKey,
    int         Credits
}
```

The algorithm assigns one credit at a time to a randomly-selected provisioner through a deterministic extraction procedure that is based on a pseudo-random number called *score*, described later.

Each provisioner can be assigned one or more credits, depending on their *stake weight* (i.e., how much active stake they have). Specifically, the extraction procedure follows a weighted distribution favoring provisioners with higher stakes. In other words, the higher the stake, the higher the probability of being assigned a credit.

To balance out this probability during the sortition process, each time a provisioner is extracted, its stake weight (within the procedure) is reduced by 1 DUSK (if the weight is lower than 1 DUSK, it is set to 0). 
Therefore, the probability for a provisioner to be extracted diminishes with every assigned credit.

As a result, eligible provisioners, on average, will participate in committees with a frequency and influence proportional to their stakes.


### Score
The extraction procedure assigns voting credits based on a pseudo-random number called *Score*, which is obtained deterministically using a cryptographic hash function.

To obtain the score, the last block's seed and the current round and step are hashed together along with the number of the credit being assigned. This hash is then interpreted as an integer to obtain the score.

Formally:
$$ Score = Int( H( Seed||Round||Step||CreditIndex ) ) $$

As a result, a unique score is generated every time a credit has to be assigned, ensuring the randomness of the extraction.

### Seed
The *Seed* value is used to ensure pseudo-randomicity and unpredictability to the *score* value used in the Sortition process.

This value is the Block structure (see [Block Structure](../../blockchain/README.md#block-structure)) and gets updated for every new block. 
Specifically, the $Seed$ for block $h$ is obtained by the block generator $BG$ signing seed in block $h-1$.
Formally: 
$$ Seed_h = Sign_{BLS}(sk_{BG}, Seed_{h-1}).$$

Therefore, only the generator of block $h$ can produce $Seed_h$. This prevents pre-calculating the score for future blocks, which in turn prevents predicting future block generators and committees.



### Procedures
The *Deterministic Sortition* algorithm can be defined by two procedures: 
- $Committee()$, which creates a *Committee* by assigning a Pool of credits to random provisioners,
and 
- $extractMember()$, which selects a provisioner at random to assign a credit.

In the following, we describe such procedures.
<!-- DONE -->

#### Committee()

Parameters:
 - $Round$: current consensus round
 - $Step$: current consensus step
 - $Seed$: the seed of block $Round-1$
 - $PoolSize$: number of voting credits in the committee
 - $Provisioners$: the set of eligible provisioners

Procedure:

$Committee(PoolSize)$:
1. Compute $TotalStakeWeight = SumAll(Provisioners.StakeWeight)$
2. Assign credits:
   - For each index $i$ ($i = 0\text{ }..\text{ }PoolSize$):
     1. Compute $ScoreHash = H( Seed||Round||Step||CreditIndex )$
     2. Compute $Score = Int(H) \text{ } \% \text{ }  TotalStake$
     3. Order $Provisioners$ by their $PubKey$ value
     4. Extract member: $M = extractCommitteeMember(Score, Provisioners) $
     5. If $M$ is not in $Committee$, add $M$ to $Committee$
     6. Assign credit to $M$
     7. Compute $Adjust$:
        ```
        if M.Stake < 1: 
            Adjust = M.Stake
        else: Adjust = 1
        ```
     8. Subtract $Adjust$ from $M.Stake$
     9. Subtract $Adjust$ from $TotalStakeWeight$
     10. If $TotalStakeWeight$ reaches 0, output $Committee$  <!-- TODO: explain this  -->


<p><br></p>

<!-- DOING : -->
#### extractMember()

Parameters:
 - $Score$: pseudo-random value for current credit assignation
 - $Provisioners$: set of eligible provisioners (ordered by their $PubKey$)

Procedure:

$extractMember(Score, Provisioners)$:
- Loop:
  - For each provisioner $P$ in $Provisioners$:
      1. If $P.StakeWeight$ is higher than or equal to $Score$, output $P$
      Else:
      2. Subtract $P.StakeWeight$ from $Score$


### Pseudocode

```
Committee(round, step, seed, poolSize, provisioners):
  totalWeight = sumStakeWeights(provisioners)

  committee = []
  for i from 0 to (poolSize-1)
    /* Compute Score */
    scoreHash = Hash(seed||round||step||i)     // Compute `scoreHash`
    score = Int(scoreHash) % totalWeight       // Compute `score` 
    
    /* Extract provisioner */
    provisioners = orderByKey(provisioners)
    m = extractMember(provisioners, score)     // Extract a member `m`
    if committee[m] == nil:                    // Add `m` to `committee`
        committee.add(m)
    committee[m].credits++                     // Assign a credit to `m`
    
    /* Adjust stake weights */
    if m.stake < 1:
      adjust = m.stake
    else:
      adjust = 1
    m.stake -= adjust               // Adjust `m`'s stake weight
    totalWeight -= adjust           // Adjust total total stake weight
    if totalWeight == 0: break      // If we "consumed" all stake weight, stop

  return votingCommittee


extractMember(provisioners, score):
    loop:                               // Loop until a provisioner is extracted
      for p in provisioners:            // For each provisioner `p`
        if p.stake >= score:            // If `p`'s stake is higher than score
          return p                      // extract `p`
        else:                           // Otherwise,
          score = score - p.stake       // Subtract `p`'s stake from `score`

```
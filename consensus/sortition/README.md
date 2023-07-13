# Deterministic Sortition
*Deterministic Sortition* is the non-interactive process used in the [Succinct Attestation](../README.md) protocol to select the *Block Generator* during the [*Attestation*](../attestation/README.md) phase, and the members of the *Voting Committees* during the [*Reduction*](../reduction) phases.

## Voting Committees
<!-- TODO: move this to Consensus main README ? -->
<!-- TODO: use weight instead of influence? -->
<!-- TODO: Change voting credits to votes? -->
A *Voting Committee* is an array of Provisioners entitled to cast votes in a specific Reduction step. Provisioners is in a committee are called *members* of such committee. 
In a given committee, each member has a certain *influence*, which is defined as the number *voting credits* (i.e., the votes it can cast) that it has been assigned by the *Deterministic Sortition* algorithm (denoted as $DS$). The total amount of credits in a committee is referred to as *Credit Pool*.

Formally, we define a member of a committee $C$ as:

$$ M^C = (pk, credits), $$ 

where $pk$ is the public key of the provisioner, and $credits$ is the number of voting credits assigned to such provisioner by the $DS$ algorithm.

Formally, a Voting Committee is defined as:

$$ C_{R}^{S} = [M_0^C,\dots,M_{n-1}^C] = DS(R,S,PS),$$

where $R$ and $S$ are the consensus round and step, respectively (see [Consensus Workflow](../README.md#workflow)), and $PS$ is the Pool Size of the committee.

Note that $C$ is ordered by insertion (a Provisioner is added to the committee when being assigned its first credit). That is, the first provisioner to be added to the list by $DS$ will have the lowest index, and the last to be inserted will have the highest. 

For the sake of readability, we use the following notation:
- $C[i] = M_i^C$ denotes the $i\text{th}$ member of $C$ 
- $C[pk] =  M_i^C : M_i^C.pk = pk$ (or $nil$ if $pk$ is not the committee)

### Reduction Committees
Voting Committees in the Reduction steps have a credit pool of size $VCPool$, which is a global consensus parameter (see [Consensus Parameters](../README.md#parameters)), and is currently set to $64$.

During a [Reduction](../reduction/) step, votes from a given member are multiplied by its influence in the committee. For instance, if a member has 3 credits, his vote will be counted 3 times.

Hence, the $VCPool$ parameter determines the maximum number of members in a committee, and, indirectly, the degree of distribution of the Reduction voting process.

### Block Generator Extraction
To extract a block generator, a one-member committee is created, using the $Deterministic Sortition$ procedure with $PS = 1$, that is, by assigning a single credit. The provisioner being assigned such credit becomes the block generator for the iteration.

Formally, the block generator for round $R$ and step $S$ is defined as:

$$ BG_R^i = DS(R,S,1).$$


## Algorithm Overview
The *Deterministic Sortition* ($DS$) algorithm generates a voting committee by assigning the credits of credit pool to eligible provisioners (see [Eligibility](../README.md#participants)).

The algorithm takes as inputs the round number $R$ and step number $S$, and outputs the corresponding committee.
It also uses the provisioner set $\mathcal{P}$, the block produced in the previous round $B_{R-1}$, and the credit pool size $PS$.

The algorithm assigns one credit at a time to a randomly-selected provisioner using the *Deterministic Extraction* ($DE$) algorithm, which is based on a pseudo-random number called [*score*](#score), described below.

Each provisioner can be assigned one or more credits, depending on their active *stake*. Specifically, the extraction algorithm follows a weighted distribution that favors provisioners with higher stakes. In other words, the higher the stake, the higher the probability of being assigned a credit.

To balance out this probability each Provisioner weight is initially set to the value of its stake, and then reduced by 1 DUSK each time the Provisioner is assigned a credit (if the weight is lower than 1 DUSK, it is set to 0). This way, the probability for a provisioner to be extracted diminishes with every assigned credit.

As such, on average, eligible Provisioners will participate in committees with a frequency and influence proportional to their stakes.


### Score
The extraction procedure assigns voting credits based on a pseudo-random number called *score*, which is generated deterministically using the *SHA3-256* [[1](#references)] hash function.

Formally:

$$ Score_{R}^{S} = Int( H_{SHA3-256}( Seed_{R-1}||R||S||i ) ) \mod W,$$

where $R$ and $S$ are the consensus round and step, $Seed_{R-1}$ is the $Seed$ of the previous block ($B_{R-1}$), $i$ is the number of the credit being assigned (e.g., $i=3$ represents the third credit to assign), and $W$ is the total *stake weight* of provisioners at iteration $i$ of $DS$. $Int()$ denotes the integer interpretation of the hash value.

The use of a hash value based on the above parameters allows to generate a unique score value for each single credit being assigned, ensuring the randomness of the extraction process.
The use of the modulo operation guarantees a Provisioner is extracted within a single iteration over the provisioner set (see the $DE$ algorithm)


### Seed
The *Seed* value is used to ensure the pseudo-randomicity and unpredictability of the score.

This value is included in the block header (see [Block Structure](../../blockchain/README.md#header-structure)) and gets updated in each block. 
Specifically, the $Seed$ of block $B_h$ is set to the signature of the block generator $BG$ on the seed of block $B_{h-1}$.

Formally: 

$$ Seed_h = Sign_{BLS}(sk_{BG_h}, Seed_{h-1}).$$

As such, $Seed_h$ can only be computed by the generator of block $B_h$. This prevents pre-calculating the score for future blocks, which in turn prevents predicting future block generators and committees.

Note that $Seed_0$ is a randomly-generated value contained in the genesis block $B_0$.

### Algorithm
We describe the *Deterministic Sortition* algorithm through the $DS$ procedure, which creates a *Voting Committee* by pseudo-randomly assigning *credits* of a *Credit Pool* to eligible provisioners using the *Deterministic Extraction* algorithm, described through the $DE$ procedure.

In the following, we describe both $DS$ and $DE$ first as natural-language algorithm and then as formal procedure.

#### Deterministic Sortition (DS)

*Parameters*:
 - $R$: consensus round
 - $S$: consensus step
 - $PS$: credit pool size
 - $W^P$: stake *weight* of provisioner $P$
 - $W$: total stake weight

*Algorithm*:
1. Set each provisioner weight $W^P$ to its stake
2. Set total weight to the sum of all $W^P$
3. Assign credit $i$ as follows:
   1. Compute $Score$
   2. Order provisioners by ascending public key
   3. Extract provisioner $P$ with $DE(Score, \mathcal{P})$
   4. If not in committee $C$, add $P$ to $C$
   5. Add credit to $P$
   6. Subtract 1 DUSK from $P$'s weight $W^P$
   7. Subtract 1 DUSK from total weight $W$
   8. If $W$ reaches 0, output the partial committee $C$
4. Output committee $C$

*Procedure*:

$DS(R, S, PS)$:
- $W^P = P.Stake, \forall P \in \mathcal{P}^R$
- $W = \sum W_{P}, \forall P \in \mathcal{P}^R$
- $\text{for } i = 0\text{ }\dots\text{ }PS{-}1$:
   - $Score_i^{R,S} = Int(H_{SHA3-256}( Seed_{R-1}||R||S||i)) \mod W$
   - $\mathcal{P}_R^O = SortByPK(\mathcal{P}^R)$
   - $P = DE(Score_i^{R,S}, \mathcal{P}_R^O)$
   - $if \text{ } C[P] \ne nil$ :
      - $C[P].credits = C[P].credits + 1$
   - $else$ :
      - $C[|C|+1] = (P, 1)$
   
   - $if \text{ } W_{P} < 1 \text{ }$ :
      - $A = W_{P}$
   - $else$:
      - $\text{ } A = 1$
   - $W_{P} = W_{P} - A$
   - $W = W - A$
   - $if \text{ } W \le 0$
       - $\text{ output } C$
- $\text{output } C$


<p><br></p>

#### Deterministic Extraction (DE)

*Parameters*:
 - $Score$: score for current credit assignation
 - $\mathcal{P}^O$: ordered set of eligible provisioners

*Algorithm*:
  1. For each provisioner $P$ in $\mathcal{P}^O$
     1. if $P$'s weight is higher than $Score$, output $P$
     2. otherwise, subtract $P$'s weight from $Score$

*Procedure*:

$DE(Score_i^{R,S}, \mathcal{P}^O)$:
<!-- - $loop$ :  This line should be unnecessary, since the for loop is guaranteed to extract -->
   - $for \text{ } P = \mathcal{P}^O[0] \dots \mathcal{P}^O[|\mathcal{P}|-1]$ :
      - $if \text{ } W_{P} \ge Score$
         - $\text{ output } P$
      - $else$
         - $Score = Score - W_{P}$

<!-- Note that the outer $loop$ means that if the $for$ loop ends (i.e. no provisioner was extracted), it starts over with $j=0$. -->


<!-- ### Pseudocode

```
DS(round, step, seed, poolSize, provisioners):
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

``` -->

<!-- REFS -->
## References
<!-- TODO: mv to Consensus README. SHA3 is also used for the block header -->
<a name="rsha3"></a>
[1]: "SHA-3 Standard: Permutation-Based Hash and Extendable-Output Functions", Boneh et al., 2004, [https://doi.org/10.6028/NIST.FIPS.202](https://doi.org/10.6028/NIST.FIPS.202)

[2]: "Short Signatures from the Weil Pairing", Dworkin, 2015, [https://doi.org/10.1007%2Fs00145-004-0314-9](https://doi.org/10.1007%2Fs00145-004-0314-9)
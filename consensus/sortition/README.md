# Deterministic Sortition
*Deterministic Sortition* is the non-interactive process used in the [Succinct Attestation](../README.md) protocol to select the *Block Generator* during the [*Attestation*](../attestation/README.md) phase, and the members of the *Voting Committees* during the [*Reduction*](../reduction) phases.

## Voting Committees
<!-- TODO: move this to Consensus main README ? -->
<!-- TODO: use weight instead of influence? -->
<!-- TODO: Change voting credits to votes? -->
A *Voting Committee* is an array of Provisioners entitled to cast votes in a specific Reduction step. Provisioners is in a committee are called *members* of such committee. 
In a given committee, each member has a certain *influence*, which is defined as the number *voting credits* (i.e., the votes it can cast) that it has been assigned by the *Deterministic Sortition* algorithm (denoted as $DS$). The total amount of credits in a committee is referred to as *Credit Pool*.

Formally, we define a member of a committee $C$ as:

$$ m^\mathcal{C} = (pk, Credits), $$ 

where $pk$ is the public key of the provisioner, and $Credits$ is the number of voting credits assigned to such provisioner by the $DS$ algorithm.

Formally, a Voting Committee is defined as:

$`$ \mathcal{C}_{r}^{s} = [m_0^\mathcal{C},\dots,m_{n}^\mathcal{C}],$`$

where $r$ and $s$ are the consensus round and step, respectively (see [Consensus Workflow](../README.md#workflow)), and $ps$ is the Pool Size of the committee.

Note that $C$ is ordered by insertion (a Provisioner is added to the committee when being assigned its first credit). That is, the first provisioner to be added to the list by $DS$ will have the lowest index, and the last to be inserted will have the highest. 

In the rest of the consensus documentation, we will use the following notation:
- $m_i^\mathcal{C}$ denotes the $i\text{th}$ member of committee $\mathcal{C}$:
  - $m_i^\mathcal{C} = \mathcal{C}[i]$
- $i_m^\mathcal{C}$ denotes the index $i$ of member $m$ in committee $\mathcal{C}$. Formally:
  - $i_m^\mathcal{C} = i : m_i^\mathcal{C} = m$

- $pk_m$ denotes the public key of member $m$ (i.e., $m.pk$):
  - $pk_m = m.pk$
- We say a provisioner with public key $pk$ is in a committee $\mathcal{C}$ if such committee contains a member with public key $pk$. Formally:
  - $pk \in \mathcal{C} \text{ }if\text{ } \exists \text{ } m^\mathcal{C} :   pk_m{=}pk$
<!-- - $\mathcal{C}[pk]$ denotes the member $m$ corresponding to public key $pk$. Formally: -->
- $m_{pk}^\mathcal{C}$ denotes the member of $\mathcal{C}$ with publick key $pk$.
  - $m_{pk}^\mathcal{C} = m : m \in \mathcal{C} \wedge pk_m{=}pk$

### Reduction Committees
Voting Committees in the Reduction steps have a credit pool of size $VCPool$, which is a global consensus parameter (see [Consensus Parameters](../README.md#parameters)), and is currently set to $64$.

During a [Reduction](../reduction/) step, votes from a given member are multiplied by its influence in the committee. For instance, if a member has 3 credits, his vote will be counted 3 times.

Hence, the $VCPool$ parameter determines the maximum number of members in a committee, and, indirectly, the degree of distribution of the Reduction voting process.

### Block Generator Extraction
To extract a block generator, a one-member committee is created, using the $Deterministic Sortition$ procedure with $ps = 1$, that is, by assigning a single credit. The provisioner being assigned such credit becomes the block generator for the iteration.

Formally, the block generator for round $r$ and step $s$ is defined as:

$$ G_r^i = DS(r,s,1).$$


## Algorithm Overview
The *Deterministic Sortition* ($DS$) algorithm generates a voting committee by assigning the credits of credit pool to eligible provisioners (see [Eligibility](../README.md#participants)).

The algorithm takes as inputs the round number $r$ and step number $s$, and outputs the corresponding committee.
It also uses the provisioner set $\mathcal{P}$, the block produced in the previous round $\mathcal{B}_{r-1}$, and the credit pool size $ps$.

The algorithm assigns one credit at a time to a randomly-selected provisioner using the *Deterministic Extraction* ($DE$) algorithm, which is based on a pseudo-random number called [*score*](#score), described below.

Each provisioner can be assigned one or more credits, depending on their active *stake*. Specifically, the extraction algorithm follows a weighted distribution that favors provisioners with higher stakes. In other words, the higher the stake, the higher the probability of being assigned a credit.

To balance out this probability each Provisioner weight is initially set to the value of its stake, and then reduced by 1 DUSK each time the Provisioner is assigned a credit (if the weight is lower than 1 DUSK, it is set to 0). This way, the probability for a provisioner to be extracted diminishes with every assigned credit.

As such, on average, eligible Provisioners will participate in committees with a frequency and influence proportional to their stakes.


### Score
The extraction procedure assigns voting credits based on a pseudo-random number called *score*, which is generated deterministically using the *SHA3-256* [[1](#references)] hash function.

Formally:

$$ Score_{r}^{s} = Int( H_{SHA3-256}( Seed_{r-1}||r||s||c ) ) \mod W,$$

where $r$ and $s$ are the consensus round and step, $Seed_{r-1}$ is the $Seed$ of the previous block ($\mathcal{B}_{r-1}$), $c$ is the number of the credit being assigned (e.g., $c=3$ represents the third credit to assign), and $W$ is the total *stake weight* of provisioners when assigning credit $c$ during $DS$. $Int()$ denotes the integer interpretation of the hash value.

The use of a hash value based on the above parameters allows to generate a unique score value for each single credit being assigned, ensuring the randomness of the extraction process.
The use of the modulo operation guarantees a Provisioner is extracted within a single iteration over the provisioner set (see the $DE$ algorithm)


### Seed
The *Seed* value is used to ensure the pseudo-randomicity and unpredictability of the score.

This value is included in the block header (see [Block Structure](../../blockchain/README.md#header-structure)) and gets updated in each block. 
Specifically, the $Seed$ of block $\mathcal{B}_h$ is set to the signature of the block generator $G$ on the seed of block $\mathcal{B}_{h-1}$.

Formally: 

$$` Seed_h = Sign_{BLS}(sk_{G_h}, Seed_{h-1}).`$$

As such, $Seed_h$ can only be computed by the generator of block $\mathcal{B}_h$. This prevents pre-calculating the score for future blocks, which in turn prevents predicting future block generators and committees.

Note that $Seed_0$ is a randomly-generated value contained in the genesis block $\mathcal{B}_0$.

### Algorithm
We describe the *Deterministic Sortition* algorithm through the $DS$ procedure, which creates a *Voting Committee* by pseudo-randomly assigning *credits* of a *Credit Pool* to eligible provisioners using the *Deterministic Extraction* algorithm, described through the $DE$ procedure.

In the following, we describe both $DS$ and $DE$ first as natural-language algorithm and then as formal procedure.

#### Deterministic Sortition (DS)

*Parameters*:
 - $r$: consensus round
 - $s$: consensus step
 - $ps$: credit pool size
 - $\mathcal{P}^r = [p_0,\dots,p_n]$: provisioner set for round $r$

*Algorithm*:
1. Start with empty committee
2. Set each provisioner's weight equal to its stake
3. Set total weight to the sum of all provisioner's weights
4. Assign credit $i$ as follows:
   1. Compute $Score$
   2. Order provisioners by ascending public key
   3. Extract provisioner $p$ with [*DE*][de]()
   4. Add $p$ to committee, if necessary
   5. Assign credit to $p$
   6. Compute $d$ as the minimum between $1$ DUSK and $p$'s weight
   7. Subtract $d$ from $p$'s weight
   8. Subtract $d$ from total weight
   9. If total weight reaches 0, output the partial committee
5. Output committee

*Procedure*:

$DS(r, s, ps)$:
1. $\mathcal{C} = \emptyset$
2. $for\text{ } p \in \mathcal{P}^r: w_p = p.Stake$
3. $W = \sum_{i=0}^{|\mathcal{P}^r|-1} w_{p_i} : p_i \in \mathcal{P}^r$
4. $for\text{ } c = 0\text{ }\dots\text{ }ps{-}1$:
   1. $Score_c^{r,s} = Int(H_{SHA3-256}( Seed_{r-1}||r||s||c)) \mod W$
   2. $\mathcal{P}_r' = SortByPK(\mathcal{P}^r)$
   3. $p_c = \text{ }$[*DE*][de]$(Score_c^{r,s}, \mathcal{P}_r')$
   4. $if \text{ } p_c \notin \mathcal{C}$ : 
       - $m_{p_c}^\mathcal{C} = (p_c,0)$ 
       - $\mathcal{C} = \mathcal{C} \cup m_{p_c}^\mathcal{C}$
   5. $m_{p_c}^\mathcal{C}.Credits = m_{p_c}^\mathcal{C}.Credits+1$
   6. $d = min(w_{p},1)$
   7. $w_{p_c} = w_{p_c} - d$
   8. $W = W - d$
   9.  $if \text{ } W \le 0$
       - $output\text{ } \mathcal{C}$
5. $output\text{ } \mathcal{C}$


<p><br></p>

#### Deterministic Extraction (DE)

*Parameters*:
 - $Score$: score for current credit assignation
 - $\mathcal{P}' = [p_0,\dots,p_n]$: array of provisioners ordered by public key

*Algorithm*:
  1. For each provisioner $p$ in $\mathcal{P}'$
     1. if $p$'s weight is higher than $Score$, 
        1. output $p$
     2. otherwise, 
        1. subtract $p$'s weight from $Score$

*Procedure*:

$`DE(Score_i^{r,s}, \mathcal{P}')`$:
<!-- - $loop$ :  This line should be unnecessary, since the for loop is guaranteed to extract -->
  <!-- 2. $for \text{ } P = \mathcal{P}^O[0] \dots \mathcal{P}^O[|\mathcal{P}|-1]$ : -->
  1. $for \text{ } p = p_0 \dots p_{|\mathcal{P}|-1}$ :  <!-- s.t. p \in \mathcal{P}'  -->
     1. $if \text{ } w_{P} \ge Score$
        1. $output\text{ } p$
     2. $else$
        1. $Score = Score - w_{p}$

<!-- Note that the outer $loop$ means that if the $for$ loop ends (i.e. no provisioner was extracted), it starts over with $j=0$. -->


## Subcommittees

<!-- TODO: BitSet -->

#### CountCredits
$CountCredits$ gets a list of members of a committee and return the cumulative amount of credits belonging to such members.

- $\textbf{p}=[pk_1,\dots,pk_n]$: set of provisioner public keys
- $\mathcal{C}$: voting committee

$Credits(\textbf{p}, \mathcal{C})$:
1. $sum = \sum_{i=0}^{n} m_{pk_i}^{\mathcal{C}}.Credits$
2. $\texttt{output } sum$


<!-- TODO: SubCommittee(C, bitset) , CountVotes(C), AggregatePKs-->

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

<!--  -->
[de]: #deterministic-extraction-de
<!-- TODO: Define BlockGenerator() and Committee() procedures -->

# Deterministic Sortition
*Deterministic Sortition* ($DS$ in short) is the non-interactive process used in the [Succinct Attestation](../README.md) protocol to select the *Block Generator* during the [*Attestation*](../attestation/README.md) phase, and the members of the *Voting Committees* during the [*Reduction*][red] phases.

## Voting Committees
<!-- TODO: move this to Consensus main README ? -->
A *Voting Committee* is an array of provisioners entitled to cast votes on the validity of a candidate block in a specific Reduction step. Provisioners in a committee are called *members* of that committee. Each member in a given committee is assigned (by the sortition process) a number of *credits* (i.e., castable vote), referred to as its *influence* in the committee.

Formally, a voting committee is defined as:

$`$ C_{r}^{s} = [m_0^C,\dots,m_n^C],$`$

where $r$ and $s$ are the consensus round and step, respectively, $n$ is the number of members in the committee, and each member $m_i^C$ is defined as:

$$ m_i^C = (pk_i, influence), $$ 

where $pk_i$ is the public key of the $i\text{th}$ provisioner to be added to the committee by the $DS$ algorithm, and $influence$ is the number of credits it is assigned.

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

### Reduction Committees
Voting Committees in the Reduction steps have a fixed number of credits that is defined by the global [consensus parameter][cp] $CommitteeCredits$, currently set to $64$.

During a [Reduction][red] step, votes from a given member are multiplied by its influence in the committee. For instance, if a member has 3 credits, his vote will be counted 3 times.

Hence, the $CommitteeCredits$ parameter determines the maximum number of members in a committee, and, indirectly, the degree of distribution of the Reduction voting process.

### Block Generator Extraction
To extract a block generator, a one-member committee is created, using the $Deterministic Sortition$ procedure with $credits = 1$, that is, by assigning a single credit. The provisioner being assigned such credit becomes the block generator for the iteration.

Formally, the block generator for round $r$ and step $s$ is defined as:

$$ G_r^i = DS(r,s,1).$$

<p><br></p>

## Algorithm Overview
The *Deterministic Sortition* ($DS$) algorithm generates a voting committee by assigning credits to [eligible](../README.md#provisioners-and-stakes) provisioners.

The algorithm takes as inputs the round number $r$ and step number $s$, and outputs the corresponding committee.
It also uses the provisioner set $\boldsymbol{P}$, the block produced in the previous round ${B}_{r-1}$, and the number of credits to assign $credits$.

The algorithm assigns one credit at a time to a randomly-selected provisioner using the *Deterministic Extraction* ($DE$) algorithm, which is based on a pseudo-random number called [*Score*](#score), described below.

Each provisioner can be assigned one or more credits, depending on their *stakes*. Specifically, the extraction algorithm follows a weighted distribution that favors provisioners with higher stakes. In other words, the higher the stake, the higher the probability of being assigned a credit.

To balance out this probability each provisioner weight is initially set to the value of its stake, and then reduced by 1 Dusk each time the provisioner is assigned a credit (if the weight is lower than 1 Dusk, it is set to 0). This way, the probability for a provisioner to be extracted diminishes with every assigned credit.

As such, on average, eligible provisioners will participate in committees with a frequency and influence proportional to their stakes.


### Score
The extraction procedure assigns credits based on a pseudo-random number called *Score*, which is generated deterministically using the *SHA3*[^1] hash function [[1](#references)].
<!-- TODO: mv SHA3 ref to main README or crypto readme -->

Formally:

$$ Score_{r}^{s} = Int( Hash_{SHA3}( Seed_{r-1}||r||s||c ) ) \mod W,$$

where $r$ and $s$ are the consensus round and step, $Seed_{r-1}$ is the $Seed$ of block $\mathsf{B}_{r-1}$, $c$ is the number of the credit being assigned (e.g., $c=3$ represents the third credit to assign), and $W$ is the total *stake weight* of provisioners when assigning credit $c$ during $DS$. $Int()$ denotes the integer interpretation of the hash value.

The use of a hash value based on the above parameters allows to generate a unique score value for each single credit being assigned, ensuring the randomness of the extraction process.
The use of the modulo operation guarantees a provisioner is extracted within a single iteration over the provisioner set (see the $DE$ algorithm).


### Seed
The *Seed* value is used to ensure the pseudo-randomicity and unpredictability of the score.

This value is included in the [block header][bh] and is updated in each block by the block generator. 
Specifically, the $Seed$ of block $`\mathsf{B}_h`$ is set to the signature of the block generator $\mathcal{G}$ on the seed of block $`\mathsf{B}_{h-1}`$.

Formally: 

$`$ Seed_h = Sign_{BLS}(sk_{G_h}, Seed_{h-1}).$`$

As such, $Seed_h$ can only be computed by the generator of block $\mathsf{B}_h$. This prevents pre-calculating the score for future blocks, which in turn prevents predicting future block generators and committees.

Note that $Seed_0$ is a randomly-generated value contained in the genesis block $\mathsf{B}_0$.

<p><br></p>

## Algorithm
We describe the *Deterministic Sortition* algorithm through the $DS$ procedure, which creates a *Voting Committee* by pseudo-randomly assigning *credits* to eligible provisioners. In turn, $DS$ uses the *Deterministic Extraction* algorithm, described through the $DE$ procedure.

In the following, we describe both $DS$ and $DE$ first as natural-language algorithm and then as formal procedure.

### Deterministic Sortition (DS)

***Parameters***
 - $r$: consensus round
 - $s$: consensus step
 - $credits$: number of credits to assign
 - $\boldsymbol{P}_r = [P_0,\dots,P_n]$: provisioner set for round $r$

***Algorithm***
1. Start with empty committee
2. Set each provisioner's weight equal to the sum of its (mature) stakes
3. Set total weight to the sum of all provisioner's weights
4. Assign credit $i$ as follows:
   1. Compute $Score$
   2. Order provisioners by ascending public key
   3. Extract provisioner $P$ with [*DE*][de]()
   4. Add $P$ to committee, if necessary
   5. Assign credit to $P$
   6. Compute $d$ as the minimum between $1$ Dusk and $P$'s weight
   7. Subtract $d$ from $P$'s weight
   8. Subtract $d$ from total weight
   9. If total weight reaches 0, output the partial committee
5. Output committee

<!-- TODO: define provisioner set as ordered by pk; then remove step 2. -->
***Procedure***

$DS(r, s, credits)$:
1. $C = \emptyset$
2. $\texttt{for } P \texttt{ in } \boldsymbol{P}_r :$
   - $w_P = \sum_{i=0}^{|\boldsymbol{Stakes}_P|-1} \boldsymbol{Stakes}_P[i]$
3. $`W = \sum_{i=0}^{|\boldsymbol{P}_r|-1} w_{P_i} : P_i \in \boldsymbol{P}_r`$
4. $\texttt{for } c = 0\text{ }\dots\text{ }credits{-}1$:
   1. $Score_c^{r,s} = Int(Hash_{SHA3}( Seed_{r-1}||r||s||c)) \mod W$
   2. $\boldsymbol{P}_r' = SortByPK(\boldsymbol{P}_r)$
   3. $P_c = \text{ }$[*DE*][de]$(Score_c^{r,s}, \boldsymbol{P}_r')$
   4. $\texttt{if } P_c \notin C$ : 
       - $m_{P_c}^C = (P_c,0)$ 
       - $C = C \cup m_{P_c}^C$
   5. $m_{P_c}^C.influence = m_{P_c}^C.influence+1$
   6. $d = min(w_{P},1)$
   7. $w_{P_c} = w_{P_c} - d$
   8. $W = W - d$
   9.  $\texttt{if } W \le 0$
       - $\texttt{output } C$
5. $\texttt{output } C$

<p><br></p>

### Deterministic Extraction (DE)

***Parameters***
 - $Score$: score for current credit assignation
 - $\boldsymbol{P} = [P_0,\dots,P_n]$: array of provisioners ordered by public key

***Algorithm***
  1. For each provisioner $P$ in $\boldsymbol{P}$
     1. if $P$'s weight is higher than $Score$, 
        1. output $P$
     2. otherwise, 
        1. subtract $P$'s weight from $Score$

***Procedure***

$`DE(Score_i^{r,s}, \boldsymbol{P})`$:
  1. $\texttt{for }  P = \boldsymbol{P}[0] \dots \boldsymbol{P}[|\boldsymbol{P}|-1]$ :
     1. $\texttt{if }  w_{P} \ge Score$
        1. $\texttt{output }  P$
     2. $\texttt{\texttt{else }}$
        1. $Score = Score - w_{P}$

<!-- Note that the outer $loop$ means that if the $for$ loop ends (i.e. no provisioner was extracted), it starts over with $j=0$. -->


## Subcommittees

<!-- TODO: BitSet -->

### CountCredits
<!-- TODO: fix \sum rendering -->
$CountCredits$ gets a list of members of a committee and return the cumulative amount of credits belonging to such members.

- $\boldsymbol{P}=[pk_1,\dots,pk_n]$: set of provisioner public keys
- $C$: voting committee

$CountCredits(\boldsymbol{P}, C)$:
1. $`sum = \sum_{i=0}^{n} m_{pk_i}^{C}.influence`$
2. $\texttt{output } sum$


<!-- TODO: SubCommittee(C, bitset) , CountVotes(C), AggregatePKs-->


<!----------------------- FOOTNOTES ----------------------->

[^1]: We will use *SHA3* to denote SHA3-256.

<!-- REFS -->
## References
<!-- TODO: mv to Consensus README. SHA3 is also used for the block header -->
<a name="rsha3"></a>
[1]: "SHA-3 Standard: Permutation-Based Hash and Extendable-Output Functions", Boneh et al., 2004, [https://doi.org/10.6028/NIST.FIPS.202](https://doi.org/10.6028/NIST.FIPS.202)

[2]: "Short Signatures from the Weil Pairing", Dworkin, 2015, [https://doi.org/10.1007%2Fs00145-004-0314-9](https://doi.org/10.1007%2Fs00145-004-0314-9)

<!------------------------- LINKS ------------------------->

[de]: #deterministic-extraction-de
[cp]: ../README.md#consensus-parameters
[red]: ../reduction/README.md
[bh]: ../../blockchain/README.md#blockheader-structure
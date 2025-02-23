# Deterministic Sortition
*Deterministic Sortition* (*DS* in short) is the non-interactive process used in the [Succinct Attestation][sa] protocol to select the *Block Generator* for the [*Proposal*][prop] phase, and the members of the *Voting Committees* for the [*Validation*][val] and [*Ratification*][rat] phases.

**ToC**
  - [Overview](#overview)
    - [Score](#score)
    - [Seed](#seed)
  - [Procedures](#procedures)
    - [Deterministic Sortition (*DS*)](#deterministic-sortition-ds)
    - [Deterministic Extraction (*DE*)](#deterministic-extraction-de)
  - [References](#references)


## Overview
The *Deterministic Sortition* ($DS$) algorithm generates a [voting committee][com] by assigning credits to [eligible][pro] provisioners.

The algorithm takes as inputs the round number $R$ and step number $S$, and outputs the corresponding committee.
It also uses the provisioner set $\boldsymbol{P}$, the block produced in the previous round ${B}_{R-1}$, and the number of credits to assign $credits$.

The algorithm assigns one credit at a time to a randomly selected provisioner using the *Deterministic Extraction* ($DE$) algorithm, which is based on a pseudo-random number called [*Score*](#score), described below.

Each provisioner can be assigned one or more credits, depending on their *stakes*. Specifically, the extraction algorithm follows a weighted distribution that favors provisioners with higher stakes. In other words, the higher the stake, the higher the probability of being assigned a credit.

To balance out this probability each provisioner weight is initially set to the value of its stake, and then reduced by 1 Dusk each time the provisioner is assigned a credit (if the weight is lower than 1 Dusk, it is set to 0). This way, the probability for a provisioner to be extracted diminishes with every assigned credit.

As such, on average, eligible provisioners will participate in committees with a frequency and power proportional to their stakes.


### Score
The extraction procedure assigns credits based on a pseudo-random number called *Score*, which is generated deterministically using the *SHA3*[^1] hash function [[1](#references)].
<!-- TODO: mv SHA3 ref to main README or crypto readme -->

Formally:

$$ Score_{R}^{S} = Int( Hash_{SHA3}( Seed_{R-1}||R||S||c ) ) \mod W,$$

where $R$ and $S$ are the consensus round and step, $Seed_{R-1}$ is the $Seed$ of block $\mathsf{B}_{R-1}$, $c$ is the number of the credit being assigned (e.g., $c=3$ represents the third credit to assign), and $W$ is the total *stake weight* of provisioners when assigning credit $c$ during $DS$. $Int()$ denotes the integer interpretation of the hash value.

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

## Procedures
We describe the *Deterministic Sortition* algorithm through the *DS* procedure, which creates a *Voting Committee* by pseudo-randomly assigning *credits* to eligible provisioners. In turn, *DS* uses the *Deterministic Extraction* (*DE*) procedure, which performs a single extraction.

### Deterministic Sortition (*DS*)

**Parameters**

 - $R$: consensus round
 - $S$: consensus step
 - $Credits$: number of credits to assign
 - $\boldsymbol{P} = [\mathcal{P}_0,\dots,\mathcal{P}_n]$: provisioner list

**Algorithm**

1. Start with empty committee
2. Set each provisioner's weight equal to the amount of its stake
3. Set total weight $W$ to the sum of all provisioner's weights
4. Assign credit $i$ as follows:
   1. Compute $Score$
   2. Order provisioners by ascending public key
   3. Extract provisioner $\mathcal{P}$ with [*DE*][de]
   4. Add $\mathcal{P}$ to committee, if necessary
   5. Assign credit to $\mathcal{P}$
   6. Compute $d$ as the minimum between $1$ Dusk and $\mathcal{P}$'s weight
   7. Subtract $d$ from $\mathcal{P}$'s weight
   8. Subtract $d$ from total weight $W$
   9. If total weight reaches 0, output the partial committee
5. Output committee

**Procedure**

$DS(R, S, Credits, \boldsymbol{P})$:
1. $\mathcal{C} = \emptyset$
2. $\texttt{for } \mathcal{P} \texttt{ in } \boldsymbol{P} :$
   - $w_\mathcal{P} = S_\mathcal{P}.Amount$
3. $`W = \sum_{i=0}^{|\boldsymbol{P}|-1} w_{\mathcal{P}_i} : \mathcal{P}_i \in \boldsymbol{P}`$
4. $\texttt{for } c = 0\text{ }\dots\text{ }credits{-}1$:
   1. $Score_c^{R,S} = Int(Hash_{SHA3}( Seed_{R-1}||S||c)) \mod W$
   2. $\boldsymbol{P}' = SortByPK(\boldsymbol{P})$
   3. $\mathcal{P}_c = \text{ }$[*DE*][de]$(Score_c^{R,S}, \boldsymbol{P}')$
   4. $\texttt{if } \mathcal{P}_c \notin \mathcal{C}$ : 
       - $m_{\mathcal{P}_c}^\mathcal{C} = (\mathcal{P}_c,0)$ 
       - $\mathcal{C} = \mathcal{C} \cup m_{\mathcal{P}_c}^\mathcal{C}$
   5. $`m_{\mathcal{P}_c}^\mathcal{C}.Power = m_{\mathcal{P}_c}^\mathcal{C}.Power+1`$
   6. $d = min(w_{P},1)$
   7. $`w_{\mathcal{P}_c} = w_{\mathcal{P}_c} - d`$
   8. $W = W - d$
   9.  $\texttt{if } W \le 0$
       - $\texttt{output } \mathcal{C}$
5. $\texttt{output } \mathcal{C}$

<p><br></p>

### Deterministic Extraction (*DE*)

**Parameters**
 - $Score$: score for current credit assignation
 - $\boldsymbol{P} = [\mathcal{P}_0,\dots,\mathcal{P}_n]$: array of provisioners ordered by public key

**Algorithm**
  1. For each provisioner $\mathcal{P}$ in $\boldsymbol{P}$
     1. if $\mathcal{P}$'s weight is higher than $Score$, 
        1. output $\mathcal{P}$
     2. otherwise, 
        1. subtract $\mathcal{P}$'s weight from $Score$

**Procedure**

$`DE(Score_i^{R,S}, \boldsymbol{P})`$:
  1. $\texttt{for }  \mathcal{P} = \boldsymbol{P}[0] \dots \boldsymbol{P}[|\boldsymbol{P}|-1]$ :
     1. $\texttt{if } w_\mathcal{P} \ge Score$
        1. $\texttt{output } \mathcal{P}$
     2. $\texttt{\texttt{else}}:$
        1. $Score = Score - w_\mathcal{P}$

<!-- Note that the outer $loop$ means that if the $for$ loop ends (i.e. no provisioner was extracted), it starts over with $j=0$. -->

<p><br></p>



<!----------------------- REFERENCES ----------------------->
## References
<!-- TODO: mv to Consensus README. SHA3 is also used for the block header -->
<a name="rsha3"></a>
[1]: "SHA-3 Standard: Permutation-Based Hash and Extendable-Output Functions", Boneh et al., 2004, [https://doi.org/10.6028/NIST.FIPS.202](https://doi.org/10.6028/NIST.FIPS.202)

[2]: "Short Signatures from the Weil Pairing", Dworkin, 2015, [https://doi.org/10.1007%2Fs00145-004-0314-9](https://doi.org/10.1007%2Fs00145-004-0314-9)


<!----------------------- FOOTNOTES ----------------------->

[^1]: We will use *SHA3* to denote SHA3-256.

<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/sortition.md  -->
[ds]: #deterministic-sortition-ds
[de]: #deterministic-extraction-de

<!-- Basics -->
[bh]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/blockchain.md#blockheader-structure

[pro]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/staking.md#provisioners-and-stakes

[com]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/attestation.md#voting-committees

<!-- Protocol -->
[sa]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md
[env]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/succinct-attestation.md#environment

[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/proposal.md
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/validation.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/protocol/steps/ratification.md

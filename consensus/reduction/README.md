# Reduction Phase
In the Reduction phase, the *candidate* block produced in the [Attestation](../attestation/) phase is validated by a subset of [Provisioners](../README.md#participants), who verify the block and cast a vote in favor or against it. The phase is divided into two steps, each of which has a committee (using [*Deterministic Sortition*][ds]) vote on the validity of the block.
If the votes of both committees reach a quorum, an $Agreement$ message broadcasted, which contains the (quorum) votes of both committees.

## Step Overview
In each reduction step, selected committee members cast votes on the candidate block, if any. A vote can be either the candidate's hash, to vote in favor, or $NIL$ to vote against. Votes are propagated through the network via $Reduction$ messages, and collected by other nodes, which accumulate them until a *quorum* is reached.

When a quorum of votes is reached, a $StepVotes$ structure is produced, containing all (and only[^1]) the quorum votes aggregated, along with a bitset of the voters with respect to the [Voting Committee](../sortition/README.md#voting-committees).

If a quorum is reached in both steps, an $Agreement$ message is produced with the $StepVotes$ structures of both steps. This message is passed to the Agreement process, which is responsible for its propagation in the network.

If any of the two steps produces a $NIL$ result, the Reduction phase will fail and a new iteration will start (i.e., a new Attestation phase will commence).
<!-- Currently, if the first step produces $NIL$, nodes still execute the second step.
This behavior should be avoided. If the goal is to spend time, just wait timeout. -->


### Reduction Message
The $Reduction$ message is used by a member of a Reduction committee to cast a vote on a candidate block. The vote is expressed by the $Header$'s $BlockHash$ field: if containing a hash, the vote is in favor of the corresponding block; if containing a $NIL$, the vote is against the candidate block of the $Round$ and $Step$ specified in the $Header$.


The message has the following structure:
| Field       | Type                  | Size      | Description           |
|-------------|-----------------------|-----------|-----------------------|
| $Header$    | [*MessageHeader*][mh] | 137 bytes | Message header        |
| $Signature$ | BLS Signature         | 48 bytes  | Signature of $Header$ |

The $Reduction$ message has a total size of 185 bytes.

#### StepVotes
The $StepVotes$ structure is produced at the end of a Reduction step and contains a quorum of votes in favor or against a candidate block.

To specify the committee members, whose vote is included in $Votes$, a bitset is used, with each bit corresponding to a committee member: if the bit is set to $1$, the corresponding member's vote is in $Votes$, otherwise it's not.

The structure is defined as follows:

| Field    | Type          | Size      | Description                       |
|----------|---------------|-----------|-----------------------------------|
| $Voters$ | BitSet        | 64 bits   | Bitset of the voters              |
| $Votes$  | BLS Signature | 48 bytes  | Aggregated $Reduction$ signatures |

Thus, the $StepVotes$ structure has a total size of 56 bytes.

Note that the 64-bit bitset is enough to represent the maximum number of members in a committee (i.e., [*VCPool*](../README.md#consensus-parameters)).


#### Agreement Message
| Field            | Type                  | Size      | Description                      |
|------------------|-----------------------|-----------|----------------------------------|
| $Header$         | [*MessageHeader*][mh] | 137 bytes | Consensus header                 |
| $Signature$      | BLS Signature         | 48 bytes  | Message signature                |
| $ReductionVotes$ | [StepVotes][sv][ ]    | 112 bytes | First and second Reduction votes |

The $Agreement$ message has total size of 297 bytes.

## Reduction Algorithm

**Parameters**:
- $\mathcal{B}^c$ : candidate block 
- $rstep$ : Reduction step number (1 or 2)
- [Consensus Parameters](../README.md#parameters)

**Environment**:
- $\mathcal{V}^{1}$ : $StepVotes$ of first Reduction
- $\mathcal{V}^{2}$ : $StepVotes$ of second Reduction
<!-- TODO?: \mathcal{V}_{r,i}^1 -->


**Algorithm**
1. Extract voting committee for the step
2. Start timeout
3. If part of the committee:
   1. If candidate is empty:
      1. Set vote $v$ to $NIL$
   2. Otherwise:
      1. Verify candidate block
      2. If block is valid, set vote $v$ to block's hash
      3. Otherwise, set $v$ to $NIL$
   3. Send $Reduction$ message with vote $v$
4. While timeout is not expired:
   1. If a $Reduction$ message $\mathcal{M}$ is received for round $r$ and step $s$:
      1. If sender is in the committee
      2. and $\mathcal{M}$'s signature is valid
         1. Propagate message
         2. Set vote $v$ to $\mathcal{M}$'s $BlockHash$ (candidate or $NIL$)
         3. Aggregate $v$ to corresponding aggregated signature
         4. Add sender to corresponding voters bitset
         5. If aggregated $v$ votes reached a quorum
            1. If this is the first Reduction:
               1. Set $\mathcal{V}^1$ to aggregated $v$
               2. Start second Reduction
            2. Otherwise:
               1. Set $\mathcal{V}^2$ to aggregated $v$
               2. Create $Agreement$ message with both $StepVotes$
               3. Broadcast message
               4. Start new consensus iteration
 5. If timeout expired:
    1. Increase Reduction timeout
    2. If it's the first Reduction:
       1. Set $\mathcal{V}^1$ to $NIL$
       2. Start the second Reduction <!-- This is kind of useless -->
    3. Otherwise:
       1. Start new consensus iteration

**Procedure**

$RunReduction( \mathcal{B}^c, rstep )$:
- $\sigma^{\mathcal{B}^c}$ : aggregate signature for candidate
- $\boldsymbol{vbs}^{\mathcal{B}^c}$ : Voters bitset for candidate
- $\sigma^{NIL}$ : aggregate signature for vote NIL
- $\boldsymbol{vbs}^{NIL}$ : Voters bitset for NIL
1. $\mathcal{C}$ = [*DS*][dsa]$(r,s,VCPool)$
2. $\tau_{Start} = \tau_{Now}$
3. $if \text{ } pk_\mathcal{N} \in \mathcal{C}$
   1. $if \text{ } \mathcal{B}^c == NIL$
      1. $v = NIL$
   2. $else$
      1. $isValid$ = [*VerifyCandidate*](#verifycandidate)($\mathcal{B}^c$)
      2. $if \text{ } isValid$: $v = H_{}(\mathcal{H}_{SHA3}^{\mathcal{B}^c})$
      3. $else : v = NIL$
   3. $\mathcal{M}^R = $[*Msg*][msg]$(Reduction, v)$
      | Field            | Value                           | 
      |------------------|---------------------------------|
      | $Header$         | $\mathcal{H}_{\mathcal{M}^R}$   |
      | $Signature$      | $\sigma_{\mathcal{M}^R}$        |

      <!-- Add | $Vote$ | $v$ | -->

   4. [*SendReduction*](#sendreduction)($v$)
4. $while \text{ } \tau_{now} \le \tau_{Start}+\tau_{Reduction_1}$ :
   1. $if \text{ } \mathcal{M}^R =$ [*Receive*][msr]$(Reduction,r,s) \ne NIL$
      1. $if \text{ } pk_{\mathcal{M}^R} \in \mathcal{C}$
      <!-- TODO?: S = pk_M  set "Sender" -->
      2. $and \text{ }$*VerifySignature*$(\mathcal{M}^R) = true$
         1. [*Propagate*][kad]()($\mathcal{M}^R$)
         2. $v = \mathcal{H}_{\mathcal{M}^R}.BlockHash$
         3. *AggregateSig*$(\sigma^v, \sigma_{\mathcal{M}^R})$
         4. $\boldsymbol{vbs}^{v}[\mathcal{C}[pk_{\mathcal{M}^R}]] = 1$
         5. $if$ *countSetBits*$(\boldsymbol{vbs}^v) \ge Quorum$
            1. $if \text{ } rstep = 1$
               1. $\mathcal{V}^1 = (\sigma^v, \boldsymbol{vbs}^v)$
               2. *RunReduction*$(\mathcal{B}^c, 2)$
            2. $else$
               1. $\mathcal{V}^2 = (\sigma^v, \boldsymbol{vbs}^v)$
               2. $\mathcal{M}^A =$ [*Msg*][msg]$(Agreement, [\mathcal{V}^1,\mathcal{V}^2])$
               <!-- (\mathcal{H}_\mathcal{M},\sigma_\mathcal{M},[\mathcal{V}^1,\mathcal{V}^2])$ -->
                  <!-- $Agreement$ -->
                  | Field            | Value                           | 
                  |------------------|---------------------------------|
                  | $Header$         | $\mathcal{H}_\mathcal{M}$       |
                  | $Signature$      | $\sigma_\mathcal{M}$            |
                  | $ReductionVotes$ | $[\mathcal{V}^1,\mathcal{V}^2]$ |

               3. *Broadcast*$(\mathcal{M}^A)$
               4. *NewIteration*$()$

 5. $if \text{ } \tau_{Now} \gt \tau_{Start}+\tau_{Reduction_{rstep}}$
    1. *IncreaseTimeout*$(\tau_{Reduction_{rstep}})$
    2. $if \text{ } rstep = 1$
       1. $\mathcal{V}^1 = \emptyset$
       2. *RunReduction*$(\mathcal{B}^c, 2)$
    3. $else$
       1. *NewIteration*$()$

---

#### VerifyCandidate
*VerifyCandidate* returns $true$ if all candidate block header fields check out with respect to the current state (i.e., the $TIP$) and the candidate block's transactions. Otherwise it returns $false$.

**Parameters**  :
  - $\mathcal{B}^c$: candidate block
  - [Consensus Parameters](../README.md#parameters)

**Algorithm**:
1. If $Version$ is $0$
2. and $Height$ is $TIP$'s height plus 1
3. and $Hash$ is the header's hash
4. and $PrevBlock$ is $TIP$
5. and $Timestamp$ is higher than $TIP$'s one
6. and $Timestamp$ is not higher than $TIP$'s timestamp plus $MaxBlockTime$
7. and transaction root corresponds to the transaction set
8. and state hash corresponds to the result of the state transition
   1. Output $true$
9. Otherwise,
   1.  Output $false$

**Procedure**:

$VerifyCandidate(\mathcal{B}^c)$:
1. $if \text{ } \mathcal{B}^c.Version = 0$ 
2. $and \text{ } \mathcal{B}^c.Height = \mathcal{B}^{tip}.Height$
3. $and \text{ } \mathcal{B}^c.Hash = \mathcal{H}_{\mathcal{B}^c}$
4. $and \text{ } \mathcal{B}^c.PreviousBlock = \mathcal{B}^{tip}.Hash$
5. $and \text{ } \mathcal{B}^c.Timestamp \ge \mathcal{B}^{tip}.Timestamp$
6. $and \text{ } \mathcal{B}^c.TransactionRoot = MerkleTree(\mathcal{B}^c.Transactions).Root$
7. $and \text{ } \mathcal{B}^c.StateRoot \ge StateTransition(\mathcal{B}^c.Transactions)$
   1. $output \text{ } true$
8. $else$
   1. $output \text{ } false$


<!-- FOOTNOTES -->
[^1]: This means that when creating a $StepVotes$ for vote $v$ only related votes are included.

<!-- LINKS -->
[ds]: ../sortition/README.md
[dsa]: ../sortition/README.md#deterministic-sortition-ds
[mh]: ../README.md#message-header
[msr]: ../README.md#send-and-receive
[msg]: ../README.md#create-message
[sv]: #stepvotes
[kad]: ../../network
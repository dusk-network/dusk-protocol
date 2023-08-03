# Reduction Phase
In the Reduction phase, the *candidate* block produced in the [Attestation](../attestation/) phase is validated by a subset of [provisioner](../README.md#participants), who verify the block and cast a vote in favor or against it. The phase is divided into two steps, each of which has a committee (using [*Deterministic Sortition*][ds]) vote on the validity of the block.
If the votes of both committees reach a quorum, an $\mathsf{Agreement}$ message broadcasted, which contains the (quorum) votes of both committees.

## Step Overview
In each reduction step, selected committee members cast votes on the candidate block, if any. A vote can be either the candidate's hash, to vote in favor, or $NIL$ to vote against. Votes are propagated through the network via $\mathsf{Reduction}$ messages, and collected by other nodes, which accumulate them until a *quorum* is reached.

When a quorum of votes is reached, a $StepVotes$ structure is produced, containing all (and only[^1]) the quorum votes aggregated, along with a bitset of the voters with respect to the [Voting Committee](../sortition/README.md#voting-committees).

If a quorum is reached in both steps, an $\mathsf{Agreement}$ message is produced with the $StepVotes$ structures of both steps. This message is passed to the Agreement process, which is responsible for its propagation in the network.

If any of the two steps produces a $NIL$ result, the Reduction phase will fail and a new iteration will start (i.e., a new Attestation phase will commence).
<!-- Currently, if the first step produces $NIL$, nodes still execute the second step.
This behavior should be avoided. If the goal is to spend time, just wait timeout. -->


### Reduction Message
The $\mathsf{Reduction}$ message is used by a member of a Reduction committee to cast a vote on a candidate block. The vote is expressed by the $Header$'s $BlockHash$ field: if containing a hash, the vote is in favor of the corresponding block; if containing a $NIL$, the vote is against the candidate block of the $Round$ and $Step$ specified in the $Header$.


The message has the following structure:
| Field       | Type                  | Size      | Description           |
|-------------|-----------------------|-----------|-----------------------|
| $Header$    | [*MessageHeader*][mh] | 137 bytes | Message header        |
| $Signature$ | BLS Signature         | 48 bytes  | Signature of $Header$ |

The $\mathsf{Reduction}$ message has a total size of 185 bytes.

#### StepVotes
The $StepVotes$ structure is produced at the end of a Reduction step and contains a quorum of votes in favor or against a candidate block.

To specify the committee members, whose vote is included in $Votes$, a bitset is used, with each bit corresponding to a committee member: if the bit is set to $1$, the corresponding member's vote is in $Votes$, otherwise it's not.

The structure is defined as follows:

| Field    | Type          | Size      | Description                       |
|----------|---------------|-----------|-----------------------------------|
| $Voters$ | BitSet        | 64 bits   | Bitset of the voters              |
| $Votes$  | BLS Signature | 48 bytes  | Aggregated $\mathsf{Reduction}$ signatures |

Thus, the $StepVotes$ structure has a total size of 56 bytes.

Note that the 64-bit bitset is enough to represent the maximum number of members in a committee (i.e., [*CommitteeCredits*](../README.md#consensus-parameters)).


#### Agreement Message
| Field            | Type                  | Size      | Description                      |
|------------------|-----------------------|-----------|----------------------------------|
| $Header$         | [*MessageHeader*][mh] | 137 bytes | Consensus header                 |
| $Signature$      | BLS Signature         | 48 bytes  | Message signature                |
| $RVotes$ | [StepVotes][sv][ ]    | 112 bytes | First and second Reduction votes |

The $\mathsf{Agreement}$ message has total size of 297 bytes.

## Reduction Algorithm

**Parameters**:
- $\mathsf{B}^c$ : candidate block 
- $rstep$ : Reduction step number (1 or 2)
- [Consensus Parameters](../README.md#parameters)

**Environment**:
- $\mathsf{V}^{1}$ : $StepVotes$ of first Reduction
- $\mathsf{V}^{2}$ : $StepVotes$ of second Reduction
<!-- TODO?: \mathsf{V}_{r,i}^1 -->


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
   3. Send $\mathsf{Reduction}$ message with vote $v$
4. While timeout is not expired:
   1. If a $\mathsf{Reduction}$ message $\mathsf{M}$ is received for round $r$ and step $s$:
      1. If sender is in the committee
      2. and $\mathsf{M}$'s signature is valid
         1. Propagate message
         2. Set vote $v$ to $\mathsf{M}$'s $BlockHash$ (candidate or $NIL$)
         3. Aggregate $v$ to corresponding aggregated signature
         4. Add sender to corresponding voters bitset
         5. If aggregated $v$ votes reached a quorum
            1. If this is the first Reduction:
               1. Set $\mathsf{V}^1$ to aggregated $v$
               2. Start second Reduction
            2. Otherwise:
               1. Set $\mathsf{V}^2$ to aggregated $v$
               2. Create $\mathsf{Agreement}$ message with both $StepVotes$
               3. Broadcast message
               4. Start new consensus iteration
 5. If timeout expired:
    1. Increase Reduction timeout
    2. If it's the first Reduction:
       1. Set $\mathsf{V}^1$ to $NIL$
       2. Start the second Reduction <!-- This is kind of useless -->
    3. Otherwise:
       1. Start new consensus iteration

**Procedure**

$RunReduction( \mathsf{B}^c, rstep )$:
- $\sigma^{\mathsf{B}^c}$ : aggregate signature for candidate
- $\boldsymbol{vbs}^{\mathsf{B}^c}$ : Voters bitset for candidate
- $\sigma^{NIL}$ : aggregate signature for vote NIL
- $\boldsymbol{vbs}^{NIL}$ : Voters bitset for NIL
1. $C$ = [*DS*][dsa]$(r,s,CommitteeCredits)$
2. $\tau_{Start} = \tau_{Now}$
3. $\texttt{if } pk_\mathcal{N} \in C$
   1. $\texttt{if } \mathsf{B}^c == NIL$
      1. $v = NIL$
   2. $\texttt{else }$
      1. $isValid$ = [*VerifyCandidate*](#verifycandidate)($\mathsf{B}^c$)
      2. $\texttt{if } isValid : v =$ *Hash*$(\mathsf{H}_{SHA3}^{\mathsf{B}^c})$
      3. $\texttt{else } : v = NIL$
   3. $`\mathsf{M}^R = `$ [*Msg*][msg]$(Reduction, v)$
      | Field            | Value                           | 
      |------------------|---------------------------------|
      | $Header$         | $\mathsf{H}_{\mathsf{M}^R}$   |
      | $Signature$      | $\sigma_{\mathsf{M}^R}$        |

      <!-- Add | $Vote$ | $v$ | -->

   4. [*SendReduction*](#sendreduction)($v$)
4. $\texttt{while} \text{ } \tau_{now} \le \tau_{Start}+\tau_{Reduction_1}$ :
   1. $\texttt{if } \mathsf{M}^R =$ [*Receive*][msr]$(Reduction,r,s) \ne NIL$
      1. $\texttt{if } pk_{\mathsf{M}^R} \in C$
      <!-- TODO?: S = pk_M  set "Sender" -->
      2. $\texttt{and }$*VerifySignature*$(\mathsf{M}^R) = true$
         1. [*Propagate*][kad]()($\mathsf{M}^R$)
         2. $v = \mathsf{H}_{\mathsf{M}^R}.BlockHash$
         3. *AggregateSig*$(\sigma^v, \sigma_{\mathsf{M}^R})$
         4. $m = m_{pk_{\mathsf{M}^R}}$ \
            $\boldsymbol{vbs}^{v}[i_m^C] = 1$
         5. $\texttt{if}$ *countSetBits*$(\boldsymbol{vbs}^v) \ge Quorum$
            1. $\texttt{if } rstep = 1$
               1. $\mathsf{V}^1 = (\sigma^v, \boldsymbol{vbs}^v)$
               2. *RunReduction*$(\mathsf{B}^c, 2)$
            2. $\texttt{else }$
               1. $\mathsf{V}^2 = (\sigma^v, \boldsymbol{vbs}^v)$
               2. $\mathsf{M}^A =$ [*Msg*][msg]$(Agreement, [\mathsf{V}^1,\mathsf{V}^2])$
               <!-- (\mathsf{H}_\mathsf{M},\sigma_\mathsf{M},[\mathsf{V}^1,\mathsf{V}^2])$ -->
                  <!-- $\mathsf{Agreement}$ -->
                  | Field       | Value                           | 
                  |-------------|---------------------------------|
                  | $Header$    | $\mathsf{H}_\mathsf{M}$       |
                  | $Signature$ | $\sigma_\mathsf{M}$            |
                  | $RVotes$    | $[\mathsf{V}^1,\mathsf{V}^2]$ |

               3. *Broadcast*$(\mathsf{M}^A)$
               4. *NewIteration*$()$

 5. $\texttt{if } \tau_{Now} \gt \tau_{Start}+\tau_{Reduction_{rstep}}$
    1. *IncreaseTimeout*$(\tau_{Reduction_{rstep}})$
    2. $\texttt{if } rstep = 1$
       1. $\mathsf{V}^1 = \emptyset$
       2. *RunReduction*$(\mathsf{B}^c, 2)$
    3. $\texttt{else }$
       1. *NewIteration*$()$

---

#### VerifyCandidate
*VerifyCandidate* returns $true$ if all candidate block header fields check out with respect to the current state (i.e., the $TIP$) and the candidate block's transactions. Otherwise it returns $false$.

**Parameters**  :
  - $\mathsf{B}^c$: candidate block
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

$VerifyCandidate(\mathsf{B}^c)$:
1. $\texttt{if } \mathsf{B}^c.Version = 0$ 
2. $\texttt{and } \mathsf{B}^c.Height = \mathsf{B}^{Tip}.Height$
3. $\texttt{and } \mathsf{B}^c.Hash = \mathsf{H}_{\mathsf{B}^c}$
4. $\texttt{and } \mathsf{B}^c.PreviousBlock = \mathsf{B}^{Tip}.Hash$
5. $\texttt{and } \mathsf{B}^c.Timestamp \ge \mathsf{B}^{Tip}.Timestamp$
6. $\texttt{and } \mathsf{B}^c.TransactionRoot = MerkleTree(\mathsf{B}^c.Transactions).Root$
7. $\texttt{and } \mathsf{B}^c.StateRoot \ge StateTransition(\mathsf{B}^c.Transactions)$
   1. $\texttt{output } true$
8. $\texttt{else }$
   1. $\texttt{output } false$


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
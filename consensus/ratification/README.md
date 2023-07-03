# Ratification Phase
The *Ratification* phase either produces a *certificate* for a candidate block, with the votes of the [Reduction](../reduction/) phase, or accepts a certificate from the network. When this occurs, the candidate block is accepted to the chain, ending the current round.

## Phase Overview
The Ratification phase runs a loop to process new `Agreement` and `AggrAgreement` messages.

`Agreement` messages are collected per block and step number, counting each message as many times as the sender's weight in the committee of the second reduction step to which it refers to.
When the number of messages for a specific block/step reaches a quorum, a new block [Certificate][cert] is produced.

When a new block certificate is produced (due to a quorum of `Agreement` messages) or received via an `AggrAgreement` message, a new *winning block* is persisted and added to the chain. 
This effectively ends the current consensus round.

## Data Structures
#### Agreement Message
| Field        | Type                   | Size      | Description       |
|--------------|------------------------|-----------|-------------------|
| hdr          | [ConsensusHeader][hdr] | 137 Bytes | Consensus header  |
| signature    | BLS Signature          | 32 bits   | signature of hdr  |
| VotesPerStep | [StepVotes][sv][]      | 192 bits  | Reduction votes   |

#### AggrAgreement Message
| Field     | Type          | Size      | Description          |
|-----------|---------------|-----------|----------------------|
| Agreement | Agreement     | 137 Bytes | Consensus header     |
| Bitset    | byte[]        | 32 bits   | signature of hdr     |
| AggrSig   | BLS Signature |  ?         | Aggregated signature |

#### Certificate Structure
| Field             | Type          | Size    | Description                                      |
|-------------------|---------------|---------|--------------------------------------------------|
| StepOneBatchedSig | BLS Signature | ?       | Aggregated signature of first reduction votes    |
| StepTwoBatchedSig | BLS Signature | ?       | Aggregated signature of second reduction votes   |
| StepOneCommittee  | byte[]        | 64 bits | BitSet of of first reduction's quorum committee  |
| StepOneCommittee  | byte[]        | 64 bits | BitSet of of second reduction's quorum committee |

## Procedure

**Environment**
- [Consensus Parameters][cparams]
- Agreement storage:
  $S = \{A_i: A_i = \{A_i^1,..., A_i^n\}\}$, \
  where $i$ is an iteration number, and $A_i$ is an Agreement message for iteration $i$.


$\textbf{\textit{RunRatification}}( )$:
Loop:
 - Handle events:
   - **`Agreement` message $A$ received**
     - If $A.Round == round$:
       - If $A.Sender \in C_{A.Round}^{A.Step}$
          1. [*Propagate*]()($A$)
          2. $isValid$ = [*VerifyAgreement*](#verifyagreement)($A$)
          3. If $isValid$:
             - Add $A$ to $S$:
               $S.A_i = S.A_i \cup A$, s.t. $i = A.Step \div 3$
             - $Count_{i} = $ [*CountAgreements*]()($S.A_i$),
             - If $Count_{i} > Quorum$:
               - $AA$ = [*CreateAggrAgreement*](#sendaggragreement)($S.A_i$)
               - $B_W$ = [*CreateWinningBlock*](#createwinningblock)($A_i^0$)
               - Output $B_W$

   - **`AggrAgreement` message $AA$ received**
     1. Verify message
        [VerifyAggregated]()($AA.hdr, AA.AggrSig, AA.Round, AA.Step, AA.BitSet$)
     2. [*Propagate*]()($AA$)
     3. $B_W =$ [*CreateWinningBlock*]($AA.Agreement$)
     4. Output $B_W$

> Note: for the sake of readability, we omit the $.hdr$ in message references.

### Subroutines

#### VerifyAgreement
1. Verify message signature:
   - $BLS.VerifySignature(A.hdr, A.Sig, A.PubKeyBLS)$
2. if $|A.VotesPerStep| \ne 2$: output $FALSE$
3. if $A.Step > ConsensusMaxStep$: output $FALSE$
4. Verify StepVotes:
   - $SV^1 {=} A.VotesPerStep^1$; $SV^2 {=} A.VotesPerStep^2$
   - [VerifyAggregated]()($A.hdr, SV^1.Signature, A.Round, A.Step{-}1, SV^1.BitSet$)
   - [VerifyAggregated]()($A.hdr, SV^2.Signature, A.Round, A.Step, SV^2.BitSet$)

#### VerifyAggregated
$VerifyAggregated$ checks the aggregated vote reaches the quorum, and the aggregated signature is valid.

$VerifyAggregated(H, \Sigma_A, Round, Step, Bitset)$:
1. $SC = SubCommittee(C_{Round}^{{Step}}, BitSet)$
2. if $CountVotes(SC) \lt Quorum$:
   - output $FALSE$
3. $APK = BLS.AggregatePubKeys(SC)$
4. output $BLS.VerifySignature(H, APK, \Sigma_A)$


#### CountAgreements
$CountAgreements$ counts the number of `Agreement` messages for iteration $i$, with each message multiplied by the [voting credits](../sortition/README.md#voting-committees) of the message sender in the second Reduction step of the same iteration.

$CountAgreements(A):$
  - $Sum = 0$
  - $\forall M \in A$:
    - $Sum = Sum + P.Credits$,
    where $P \in C_{M.Round}^{M.Step}$ and $P.PubKey=M.Sender.PubKey$
  - Output $Sum$

#### CreateAggrAgreement
*CreateAggrAgreement* aggregates all the signatures of known `Agreement` messages for a specific iteration, generates an `AggrAgreement` message, and broadcasts it.

$CreateAggrAgreement(A_i)$:
 1. Aggregate signatures: 
    - $\Sigma_{i} = BLS.Aggregate(\{\Sigma : A.\Sigma \in A_i\}) $
 2. Create q-Committee bitset:
    - $\Beta_{i} = BitSet(\{pk : pk=A.PubKey \wedge A \in A_i\})$
 3. Create `AggrAgreement`:
    - $AA = (A_i^0, \Beta_i, \Sigma_i )$
 4. Output $AA$


#### CreateWinningBlock
*CreateWinningBlock* creates a certificate and adds it to the candidate block.

$CreateWinningBlock(A)$
  - Create `Certificate`:
    - $\Phi = (A.VotesPerStep^1, A.VotesPerStep^2)$
  - Add certificate to candidate block:
    - $CB.Certificate = \Phi$
  - Output $CB$

<!------------------------- LINKS ------------------------->
[hdr]: ../README.md#consensus-message-header
[sv]: ../reduction/README.md#stepvotes
[cert]: #certificate-structure
[cparams]: ../README.md#parameters
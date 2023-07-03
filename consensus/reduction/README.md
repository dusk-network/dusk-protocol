# Reduction Phase
In the reduction phase, the *candidate* block produced in the [Attestation](../attestation/) phase is validated by two separate committees of [Provisioners](../README.md#participants).
The phase is composed of two steps, each of which has a committee voting on the validity of the candidate block. 

## Step overview
In each reduction step, selected committee members cast votes on the candidate block. Votes can be either `nil` to vote against the block, or the block's `hash` to vote in its favor. Votes are propagated through the network via `ReductionMessage` packets, and collected by other nodes, which accumulate them until a *quorum* is reached.

When a quorum of votes is reached, a [`StepVotes`](#stepvotes) structure is produced, containing all collected votes aggregated, along with a bitset indicating the set of voters, with respect to the [Voting Committee](../sortition/README.md#voting-committees).

At the end of the second reduction step, if a quorum is reached, an `Agreement` message is produced with the `StepVotes` structures of both steps. This message is passed to the Agreement process, which is responsible for its propagation in the network.

If any of the two steps produces a `nil` result, the Reduction phase will fail and a new iteration will start (i.e., a new Attestation phase will commence).

## Data Structures

#### ReductionMessage
| Field      | Type             | Size      | Description       |
|------------|------------------|-----------|-------------------|
| hdr        | [ConsensusHeader](../README.md#consensus-message-header)  |           | Consensus header  |
| SignedHash | byte[]           | 32 bits   | signature of hdr  |


#### StepVotes

| Field     | Type   | Size      | Description          |
|-----------|--------|-----------|----------------------|
| BitSet    | uint   | 64 bits   | Bitset of the voters |
| Signature | uint   | 32 bits   | Aggregated signature |

<!-- TODO: explain BitSet -->

## Procedure

**Environment**:
- [Consensus Parameters](../README.md#parameters)
- $SV_{R1}$ : `StepVotes` of first Reduction
- $SV_{R2}$ = `StepVotes` of second Reduction

**Inputs**
- $CB$ : candidate block 
- $RStep$ : Reduction step (1 or 2) 

$\textbf{\textit{RunReduction}}( CB, RStep )$:
1. **Set step parameters** :
   - $SV_{nil}$ = `StepVotes`$(BitSet{:}\empty, Signature{:}\empty)$
   - $SV_{CB}$ = `StepVotes`$(BitSet{:}\empty, Signature{:}\empty)$
   - $VC$ = [*Committee*](../sortition/README.md#createcommittee)()
2. **Set timeout** :
    $StepTimeOut$ = $time.Now + ConsensusTimeOut$
3. **Cast vote** : \
  If $this.PubKeyBLS$ in $VC$:
    - Set $vote$:
      - If $CB == nil$:
        - $vote = nil$
      - Else: 
        - $isValid$ = [*VerifyCandidateBlock*](#verifycandidateblock)($CB$)
        - If $isValid$: $vote = CB.Hash$
        - Else: $vote = nil$
    - [*SendReductionMessage*](#sendreductionmessage)($CB.Hash$, $vote$)
1. **Process votes** :
   While $time.Now \le StepTimeOut$:
    - For each [`ReductionMessage`](#reduction-message) $M$:
      -  If $M.Round == round$ 
         && $M.step == step$
         && $M.PubKeyBLS \in VC$
         && BLS.*VerifySig*($M.PubKeyBLS$, $M.hdr$, $M.SignedHash$)
         - [*Propagate*]()($M$) <!-- TODO: add link to Kadcast -->
         - Aggregate vote:
           - If $M.BlockHash$ == $CB.Hash$ :
              - [*AggregateVote*](#aggregatevote)($SV_{CB}$, $M$)
              - $SV = SV_{CB}$
           - Else:
              - [*AggregateVote*](#aggregatevote)($SV_{nil}$, $M$)
              - $SV = SV_{nil}$
         - Check quorum:
           - If [*CountVotes*](#countvotes)($SV.BitSet$) $\ge$ $Quorum$
              - If $RStep == 1$:
                - $SV_{R1} = SV$
                - *RunReduction*( $CB$, $2$ )
              - Else:
                - $SV_{R2} = SV$
                - [*SendAgreement*]()($SV_{R1}$, $SV_{R2}$) <!-- TODO -->
                - [*RunAttestation*](../attestation/README.md)()

 2. **Process Expired TimeOut**
    If $time.Now > StepTimeOut$
      - If $RStep == 1$:
        - [*IncreaseTimeout*](../README.md#increasetimeout)($Reduction1Timeout$)
        - $SV_{R1} = SV$
        - *RunReduction*( $CB$, $2$ )
      - Else:
        - [*IncreaseTimeout*](../README.md#increasetimeout)($Reduction2Timeout$)
        - [*RunAttestation*](../attestation/README.md)

---
### Subroutines

#### VerifyCandidateBlock
Inputs: 
- $TIP$: current chain tip
- $CB$: candidate block
  
$VerifyCandidateBlock(TIP, CB)$:
  - **Check Header fields**:
    - If $Version == 0$ 
      && $CB.Height$ > $TIP.Height$
      && $Hash == Hash(header)$
      && $PreviousBlockHash == TIP.Hash$
      && $Timestamp >= TIP.Timestamp$
      && $CB.Timestamp <= TIP.Timestamp + MaxBlockTime$ [only for $CB.Height > 1$]
      && $TransactionRoot == MerkleTree(CB.Transactions).root$
      && $StateHash == CalculateStateTransition(CB.Transactions)$
      - Output $TRUE$
    - Else:
      - Output $FALSE$


> Note: for the sake of readability, we omit the $.Header$ in both $CB$ and $TIP$

#### CountVotes
`CountVotes` outputs the number of `1` bits in a BitSet.

#### SendReductionMessage
Inputs:
  - $vote$ : either $nil$ or $CB$'s hash

$SendReductionMessage(vote)$:
1. Create [`ConsensusHeader`](../README.md#consensus-message-header) $hdr$:
    | Field     | Value            |
    |-----------|------------------|
    | PubKeyBLS | $this.BLSPubKey$ |
    | Round     | $round$          |
    | Step      | $step$           |
    | BlockHash | $vote$           | 

2. Sign header:
     - $sig$ = $Sign_{BLS}(this.BLSPrivKey, hdr)$

3. Create `ReductionMessage` $M$:
    | Field      | Value |
    |------------|-------|
    | hdr        | $hdr$ |
    | SignedHash | $sig$ |

 4. [*Propagate*]()($M$) <!-- TODO: add link -->

#### AggregateVote
Inputs:
  - $SV$ : `StepVotes`
  - $M$ : `ReductionMessage`


$AggregateVote(SV, M)$:
  1. Aggregate signature :
     - BLS.*AggregateSig*($SV.Signature$, $M.SignedHash$)
  2. Set bit in $BitSet$:
     - *SetBit*($SV.BitSet$, $M.PubKeyBLS$)





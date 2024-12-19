# Attestation Workflow (Rust Implementation)
[`consensus/src/selection/step.rs`]

#### Context
**`step`**

Round Update:
**`pubkey_bls`** : Public BLS key of this node's provisioner
**`secret_key`** : Secret key of this node's provisioner
**`round`** : current consensus round
**`prevSeed`** : previous block seed
**`hash`** : previous block hash
**`prevTimestamp`** : previous block timestamp

---
Selection.run()
- **(1) If extracted, generate new block and broadcast it**
  if committee.amMember()   $\hspace{30pt}$ --> [see [Sortition]()]
  - // Generate candidate block and message
    `msg` = *generate_candidate_message(ru, step)*

  - // Broadcast NewBlock message
    *send*(`msg`)   **TODO**

  - // Register block in local state
    *collect*(`msg`)    **TODO**

- **(2) Handle queued messages for current round and step**
  - *handle_future_msgs*()    **TODO**

- **(3) Handle loop**
  - *event_loop*(`committee`, `timeout`)    **TODO**

---
fn ***generate_candidate_message*** (ru, step) :
1. `iteration` = `step` / 3 + 1;

2. // Sign the previous seed to obtain the new one
    `seed` = `node`.*sign*(`prevSeed`)

3. // Generate candidate block
    **`candidate`** = *generate_block* (`ru.seed`, `ru.round`, `ru.hash`, `iteration`, `ru.prevTimestamp`) **TODO**

4. // Create message header
    `msg_header` = 
    ```
        Header {
            pubkey_bls: ru.pubkey_bls,
            round: ru.round,
            block_hash: candidate.header.hash,
            step,
            topic: Topics::NewBlock,
        };
    ```

5. // Sign message header
    `signed_hash` = *sign*(`msg_header`, `ru.secret_key`, `ru.pubkey_bls`);

6. // Create NewBlock message
    msg = 
    ```
        Message {
            msg_header,
            NewBlock {
                prev_hash: ru.hash,
                candidate: candidate,
                signed_hash: signed_hash,
            }, 
        }
    ```



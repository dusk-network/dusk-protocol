### Parameters

**Config**
**`DUSK`** = 1_000_000_000
**`blockGasLimit`** = 5 DUSK
**`MaxBlockTime`** = 360 seconds
**`ConsensusTimeOut`** = 5 seconds 
**`ExtractionDelay`** = 3 seconds
**`MaxTxSetSize`** = 825000

**Context**
**`node`** : node running the protocol
**`queue`** : consensus event queue
**`round`** : current consensus round
**`prevHash`** : previous block hash
**`prevTimestamp`** : previous block timestamp
**`prevSeed`** : previous block seed


### Phase Struct
<!-- TODO --> TBD

### Event Queue
The consensus process maintains an event queue where messages are stored by their round/step number.

The maximum number of messages that can be stored in the queue is `maxMessages`, which is currently set to `4096`.

```go
type Queue struct {
   lock    sync.RWMutex
   entries map[uint64]map[uint8][]message.Message
   items   int
}
```

Events can be inserted in the queue using 
$\hspace{5mm}$ **`PutEvent(round, step, message)`**, 
and retrieved by round and step, using 
$\hspace{5mm}$ **`GetEvents(round, step)`**.

Multiple messages for the same round and step are stored in the order they are received.

The **`Flush`** method can be used to retrieve (and remove them from the queue) all events for a given round. In this case, the events are order by step number, but giving priority to `AggrAgreement` messages.


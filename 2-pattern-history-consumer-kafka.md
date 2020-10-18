# A pattern for history consumer with Kafka

In a [previous post](ref1), I wrote about a general approach for a service to process a backlog of messages before doing anything else.

In this post, instead of using an in-memory implementation example, I'll expand on this concept by using Apache Kafka as the event store.
To interact with Kafka, we'll be using the Go client library: [Shopify/sarama](ref2).

## Recap

The history consumer pattern will:

1. read a data source from the beginning
2. understand where it is in the topic: offset and lag
3. notify others when it catches up

To be able to notify other coroutines when the consumer group catches up, we must take note of how many messages
there are to consume.
In Kafka, consumers are organised in consumer groups, and they share some properties, such as the *offset*
towards a *topic*.
You can think of a topic as the _stream_ or _dataset_ where similar messages are placed.

The offset is a number that marks a position in the topic.
There are two important offsets: the one of the next message to be consumed, and the one to be produced.
The difference between this is called the consumer group *lag*.

Allow me to (try to) illustrate:
```
topic: [x x x x . . . . . .]
                ^         ^^
                C         LP
                |---lag---|

C: offset of the next message to be Consumed
L: offset of the last message in the topic
P: offset of the next message to be Produced
```

We're interested in the *offset of the next message to be produced*, let's call this our target.
Regardless of where a consumer is in the topic, if we take note of the target when it connects to the Kafka
broker, then on each message consumed, we can check if the offset of that message has reached the target.
That is when we know the consumer has caught up.

Note that it's important to capture the target before starting consumption because most likely there
will be new messages being produced as the consumption happens.

Since there is a fairly large chuck of code to mention, most of the explanations will be given as we go, via code comments.
Let's do this!

### Setting up

```go
// HistoryConsumer is the interface describing the operations available on a consumer.
type HistoryConsumer interface {
	// Close will close the underlying messaging connections
	io.Closer
	// ReadHistory will start consuming messages from the underlying
	// messaging system, and signal the reaching of the "end of the queue"
	// via the channel exposed on the CaughtUp method.
	ReadHistory(context.Context, HandleFunc) error
	// CaughtUp will be closed when the last message that was observed at
	// process start time has been processed.
	CaughtUp() <-chan struct{}
}

// HandleFunc is expected to be provided to the consumer, and will be invoked
// for every message consumed. If the error returned is nil, the message will
// be marked as consumed to the messaging system and not be delivered again.
type HandleFunc func(context.Context, []byte) error

// KafkaConfig encapsulates the field to configure a kafka consumer.
type KafkaConfig struct {
	Version       string   // kafka broker version
	Addresses     []string // kafka broker addresses
	ConsumerGroup string   // consumer group name
	Topic         string   // topic name to consume
}

type kafkaConsumer struct {
	config       KafkaConfig
	handler      HandleFunc

	saramaConfig *sarama.Config
	client       sarama.Client

	// channel to be closed when it catches up
	caughtUpSig  chan struct{}
	caughtUp     bool

	// TODOBLOG: explain this
	// groupCtxLock sync.RWMutex
	// groupCancel  context.CancelFunc
}
```

### Reading from beginning

Sarama provides plenty of configuration options to connect to a Kafka broker.
For our goal, the important ones to set are the following:

```go
// NewKafka constructs a new history consumer.
func NewKafka(config KafkaConfig, handler HandleFunc) HistoryConsumer {
	// build a new sarama config
	sc := sarama.NewConfig()

	// set the version or a default
	version, err := sarama.ParseKafkaVersion(config.Version)
	if err != nil {
		version = sarama.V2_0_1_0
	}
	sc.Version = version

	// start reading from the beginning of the topic
	// note this will have no effect if the broker has this consumer group offset previously committed
	sc.Consumer.Offsets.Initial = sarama.OffsetOldest
	// for how long should the broker remember the consumer group offset
	// we want this to be low since when reconnecting we want the offset to be the oldest
	// can't be zero as that will default to the broker's config:
	// https://kafka.apache.org/documentation/#offsets.retention.minutes
	sc.Consumer.Offsets.Retention = time.Second * 1

	// init the sarama client
	client, err := sarama.NewClient(config.Addresses, sc)
	if err != nil {
		return err
	}

	// build our kafka history consumer struct
	k := &kafkaConsumer{
		config:       config,
		handler:      handler,

		saramaConfig: sc,
		client:       client,

		caughtUpSig:  make(chan struct{}),
		caughtUp:     false,
	}

	// Fetch the latest info about consumer group offset and lag
	// TODO: implement this (see next section)

	return k
}
```


### Getting info about consumer group offset

When our consumer connects to Kafka, it can inspect the state of the consumer group to determine
how many messages there are in the topic and the offset *of the next message to be produced*.

In Kafka a topic can be organised into one or more partitions. We won't go deep into the subject of
partitions. For now it is enough to know a consumer group has an offset for each partition.

```go
// FetchOffsetsResponse is a map of partition ids to their offsets
type FetchOffsetsResponse map[int32]int64

// FetchNewestOffsets determines the current consumer group offsets, per partition, for the provided topic
func FetchNewestOffsets(cl sarama.Client, topic string) (FetchOffsetsResponse, error) {
	// refresh the metadata for the topic
	err := cl.RefreshMetadata(topic)
	if err != nil {
		return nil, err
	}

	// get the topic partitions
	parts, err := cl.Partitions(topic)
	if err != nil {
		return nil, err
	}

	var resp FetchOffsetsResponse = make(map[int32]int64)
	for _, part := range parts {
		// get the offset for this topic + partition
		// OffsetNewest means the offset of the message that will be produced next
		partOffset, err := cl.GetOffset(topic, part, sarama.OffsetNewest)
		if err != nil {
			return nil, err
		}

		resp[part] = partOffset
	}

	return resp, nil
}
```

Back in `NewKafka` we can now call this and store the data in the struct:

```go
type kafkaConsumer struct {
	//...

	// the offsets of the message that will be produced next, for each partition
	offsets                    FetchOffsetsResponse
	// number of partitions that have a positive offset and potentially lag (needs to catch up)
	numTopicPartitions         uint32
	// number of partitions that have caught up, when it equals numTopicPartitions, the entire
	// consumer is considered caught up
	numTopicPartitionsCaughtUp uint32
}

func NewKafka(config KafkaConfig, handler HandleFunc) HistoryConsumer {
	// ...

	// Fetch the latest info about the offsets for the next produced message, per partition
	offsets, err := FetchNewestOffsets(client, config.Topic)
	if err != nil {
		return err
	}
	// check how many partitions have an offset and potentially lag (needs to catch up)
	for _, offs := range p {
		if offs > 0 {
			k.numTopicPartitions++
		}
	}
	// No offsets to catch up to, topic might be empty - set as caught up now
	if k.numTopicPartitions == 0 {
		close(k.caughtUpSig)
		k.caughtUp = true
	}

	// ...
}
```

### Identifying when the consumer group catches up
Each time a message is consumed, we can compare the current offset with the previously recorded target.
When they match, the partition is considered caught-up. When all partitions catch up, the HistoryConsumer
can notify other coroutines of this fact.

```go
func (k *kafkaConsumer) ConsumeClaim(sess sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for msg := range claim.Messages() {
		err := k.handler(sess.Context(), msg.Value)
		if err != nil {
			return err
		}
		//
		sess.MarkMessage(msg, "")

		if k.caughtUp {
			return nil
		}

		// check if this partition has caught up
		target := k.offsets[msg.Partition] - 1
		if msg.Offset == target {
			atomic.AddUint32(&k.numTopicPartitionsCaughtUp, 1)

			// check if all partitions caught up
			if k.numTopicPartitions == atomic.LoadUint32(&k.numTopicPartitionsCaughtUp) {
				if !k.caughtUp {
					// history consumer caught up
					close(k.caughtUpSig)
					k.caughtUp = true
				}
			}
		}
	}
	return nil
}
```



---


CODE DUMPS TO REVISE:

```go
func (k *kafkaConsumer) CaughtUp() <-chan struct{} {
	return k.caughtUpSig
}

func (k *kafkaConsumer) start(ctx context.Context) error {
	group, err := sarama.NewConsumerGroupFromClient(k.config.ConsumerGroup, k.client)
	if err != nil {
		return err
	}

	go func() {
		for {
			gerr, open := <-group.Errors()
			if !open {
				return
			}
			if gerr != nil {
				if cerr, ok := gerr.(*sarama.ConsumerError); ok {
					// unrecoverable error
					if cerr.Err == sarama.ErrUnknownMemberId {
						k.groupCtxLock.RLock()
						groupCancel := k.groupCancel
						k.groupCtxLock.RUnlock()
						if groupCancel != nil {
							groupCancel()
						}
					}
				}
			}
		}
	}()

	var groupErr error
consumerGroupLoop:
	for {
		groupCtx, groupCancel := context.WithCancel(ctx)
		defer groupCancel()

		k.groupCtxLock.Lock()
		k.groupCancel = groupCancel
		k.groupCtxLock.Unlock()
		groupErr = group.Consume(groupCtx, k.config.Topics, k)

		select {
		case <-groupCtx.Done():
			break consumerGroupLoop
		default:
		}

		groupCancel()
		if groupErr != nil {
			break consumerGroupLoop
		}
	}

	closeErr := group.Close()
	if closeErr != nil {
		if groupErr == nil {
			return closeErr
		}

		return fmt.Errorf(
			"error in consumer group: %v - additional error encountered closing group: %v",
			groupErr, closeErr)
	}

	return groupErr
}

```


[ref1]:https://github.com/nunosilva800/blog/blob/master/1-pattern-history-consumer.md
[ref2]:https://github.com/Shopify/sarama

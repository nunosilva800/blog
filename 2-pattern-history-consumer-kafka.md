# A pattern for history consumer with Kafka

In a [previous post](ref1), I wrote about a general approach for a service to process a backlog of messages before doing anything else.

For this scenario, one possible solution is to have an ENV var or expose an API that will turn off or on
the production of new messages, until it is known that all the backlog of messages were consumed.

This is typically done by manual interactions:
- turning off the producer
- letting the consumer do its thing to build state
- monitor the progress of consumption
- turning on the producer when it's ready

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

type kafkaConsumer struct {
	handler      HandleFunc

	saramaConfig *sarama.Config
	client       sarama.Client

	// channel to be closed when it catches up
	caughtUpSig  chan struct{}
	// to keep track of caught up state
	caughtUp     bool
}
```

### Reading from the beginning

Sarama provides plenty of configuration options to connect to a Kafka broker.
For our goal, the important ones to set are the following:

```go
// NewKafka constructs a new history consumer for the topic, connecting to broker at the addresses
// HandleFunc is the function that will handle each message received
func NewKafka(topic string, addresses []string, handler HandleFunc) HistoryConsumer {
	// build a new sarama config
	sc := sarama.NewConfig()

	// set the kafka broker version
	sc.Version = sarama.V2_0_1_0

	// start reading from the beginning of the topic
	// note this will have no effect if the broker has this consumer group offset previously committed
	sc.Consumer.Offsets.Initial = sarama.OffsetOldest
	// for how long should the broker remember the consumer group offset
	// we want this to be low since when reconnecting we want the offset to be the oldest
	// can't be zero as that will default to the broker's config:
	// https://kafka.apache.org/documentation/#offsets.retention.minutes
	sc.Consumer.Offsets.Retention = time.Second * 1

	// init the sarama client
	client, err := sarama.NewClient(addresses, sc)
	if err != nil {
		return err
	}

	// build our kafka history consumer struct
	k := &kafkaConsumer{
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

Back to `kafkaConsumer` we can now store the data in the struct and
update `NewKafka` to call `FetchNewestOffsets`:

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

func NewKafka(topic string, addresses []string, handler HandleFunc) HistoryConsumer {
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

There are a few bits purposely not mentioned here, in order to keep it as short and concise as possible.
Namely, the code that starts consumption of the messages and the handling of them, nor the necessary
primitives to protect state shared between goroutines.

This `kafkaConsumer` struct satisfies the
interface [ConsumerGroupHandler](https://pkg.go.dev/github.com/Shopify/sarama#ConsumerGroupHandler),
and only the `ConsumeClaim` function is relevant for this post.

```go
// Process the messages received in this claim, checking the offsets
func (k *kafkaConsumer) ConsumeClaim(sess sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for msg := range claim.Messages() {
		// handle this message
		err := k.handler(sess.Context(), msg.Value)
		if err != nil {
			return err
		}
		// mark message as consumed
		sess.MarkMessage(msg, "")

		// nothing more to do if we've already caught up
		if k.caughtUp {
			return nil
		}

		// check if this partition has caught up by consuming this message
		target := k.offsets[msg.Partition] - 1
		if msg.Offset == target {
			// one more partition caught up - only one goroutine should write this at a time
			k.numTopicPartitionsCaughtUp++

			// check if all partitions caught up
			if k.numTopicPartitions == k.numTopicPartitionsCaughtUp {
				// history consumer caught up
				close(k.caughtUpSig)
				k.caughtUp = true
			}
		}
	}
	return nil
}
```

Clients of this package can then listen to when the consumer group catches up with:

```go
func (k *kafkaConsumer) CaughtUp() <-chan struct{} {
	return k.caughtUpSig
}
```

## Key takeaways:

`Shopify/sarama` gives us all the tools required for implementing the HistoryConsumer pattern in Kafka.
Given Kafka's high throughput, it's possible to develop services that consume millions of messages
to build state fairly quickly and:
- without requiring persistent storage
- without requiring manual controls to disable/enable production of new messages
- can automatically start producing new messages as soon as the consumer catches up with the topic.

[ref1]:https://github.com/nunosilva800/blog/blob/master/1-pattern-history-consumer.md
[ref2]:https://github.com/Shopify/sarama

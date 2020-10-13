# A pattern for history consumer with Kafka

In a [previous post](ref1), I wrote about a general approach for a service to process a backlog of messages before doing anything else.

In this post, instead of using an in-memory implementation example, I'll expand on this concept by using Apache Kafka as the event store.

## Recap

The history consumer pattern will

1. read a data source from the beginning
2. notify when it catches up

To be able to notify other workers (threads) when it catches up, it must take note of how many messages
there are to consume.
In Kafka, consumers are organised in consumer groups, and they share some properties, such as the *offset*
with regards to a *topic*.
You can think of a topic as the "stream" or "dataset" where similar messages are placed.

The offset is a number that marks the position in the topic of the next message to be read.
There can be any number of messages after the one at the offset, and that amount is called the (consumer group) *lag*.
So, when our consumer connects to Kafka, it can inspect the state of the consumer group to determine
how many messages there are in the topic.

Each time a message is consumed, we can compare the current offset with the previously recorded total.
When they match, it's considered caught-up.
Note that it's important to capture the total messages before starting consumption because most likely there
will be new messages being written to the topic at a regular pace.

## An implementation

To interact with Kafka, we'll be using [Shopify/sarama](ref2).

```go
// Consumer is the interface describing the operations available on a consumer.
type Consumer interface {
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
type HandleFunc func(context.Context, Message) error

// Message encapsulates the data as returned by the messaging system.
type Message interface {
	Data() []byte
}

// KafkaConfig encapsulates the field to configure a kafka consumer.
type KafkaConfig struct {
	ClientID      string
	Version       string
	FromBeginning bool
	Addresses     []string
	ConsumerGroup string
	Topics        []string
}

type kafkaConsumer struct {
	handler      HandleFunc
	config       KafkaConfig

	saramaConfig *sarama.Config
	client       sarama.Client
	offsets      FetchOffsetsResponse
	groupCtxLock sync.RWMutex
	groupCancel  context.CancelFunc
	caughtUpSig  chan struct{}
	caughtUp     bool

    numTopicPartitions         uint32
	numTopicPartitionsCaughtUp uint32
}

// NewKafka constructs a new consumer.
func NewKafka(config KafkaConfig, handler HandleFunc) Consumer {
	sc := sarama.NewConfig()
    sc.ClientID = config.ClientID

	version, err := sarama.ParseKafkaVersion(config.Version)
	if err != nil {
		version = sarama.V2_0_1_0
	}

    // start reading from the beginning of the topic
	sc.Consumer.Offsets.Initial = sarama.OffsetOldest
	sc.Consumer.Offsets.Retention = time.Second * 1
	sc.Version = version

	client, err := sarama.NewClient(config.Addresses, sc)
	if err != nil {
		return err
	}

	offsets, err := FetchNewestOffsets(client, config.Topics...)
	if err != nil {
		return err
	}

  	k := &kafkaConsumer{
		config:       config,
		saramaConfig: sc,
		caughtUpSig:  make(chan struct{}),
		caughtUp:     false,
		client:       client,
		offsets:      offsets,
		handler:      handler,
	}

	for _, p := range offsets {
		for _, offs := range p {
			if offs > 0 {
				k.numTopicPartitions++
			}
		}
	}
	// No offsets to catch up to, topic might be empty - set as caught up now
	if k.numTopicPartitions == 0 {
		close(k.caughtUpSig)
		k.caughtUp = true
	}

	return k
}

// PartitionOffsets is a map of partitions with their offsets
type PartitionOffsets map[int32]int64
// FetchOffsetsResponse is a map of topics containing partition offsets
type FetchOffsetsResponse map[string]PartitionOffsets

func FetchNewestOffsets(cl sarama.Client, topics ...string) (FetchOffsetsResponse, error) {
	err := cl.RefreshMetadata(topics...)
	if err != nil {
		return nil, err
	}

	var res FetchOffsetsResponse = make(map[string]PartitionOffsets)

	for _, topic := range topics {
		parts, err := cl.Partitions(topic)
		if err != nil {
			return nil, err
		}

		for _, part := range parts {
			partOffset, err := cl.GetOffset(topic, part, sarama.OffsetNewest)
			if err != nil {
				return nil, err
			}

			partOffsets, ok := res[topic]
			if !ok {
				partOffsets = make(map[int32]int64)
			}

			partOffsets[part] = partOffset
			res[topic] = partOffsets
		}
	}

	return res, nil
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

# A pattern for history consumer in event sourcing

Event sourcing is a popular [technique to capture changes to data as a sequence of events](ref1).

Applications interested in this data usually read (consume) the events in order and build an
accumulated representation often referred to as "the state".

It is also quite common for these applications to build the state in-memory, instead of storing it in
other slower and persistent mediums.
In this case, every time the application starts it needs to go through all the events in the log and process them.

If the amount of events is low enough and the processing time is quick enough, then this can lead
to particularly lightweight micro/nano-services with no permanent storage dependencies.
The caveat is that the service probably should hold off any work until this state is rebuilt, otherwise
it may lack information it needs to correctly function.
For example, it might publish events that are already in the log, but since it has not _caught up_
with it entirely, it publishes them again, leading to duplicates, corrupting the log, etc.

In a single process or single threaded system this is a trivial problem to solve.
Simply ensure that the application first consumes all of the log, and only after that it can start doing its thing.

But that is not how modern applications are built.
Even if it takes just a few seconds to process all of the log,
users expect the application to be responsive during that time.
For example, service health and _readyness_ checks should respond accordingly, or
an API required by a user-facing front-end app can have a loading indication
while processing is done.


## Signalling when it catches up

A pattern that I've come across and grown fond of is for the thread (or group of threads) that is rebuilding
state from the log, to signal to others that depend on the state, that it is has caught up, ergo,
the state is ready.

You might already be thinking how this can be achieved with most concurrency primitives, such as locks or
semaphores, and yes, this is a good way to handle the problem.

An example of this that uses Go's channels would look like the following:

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

type StateBuilder struct {
	caughtUpSig chan struct{}
}

func (sb *StateBuilder) Start() {
	// sleep for some random time to simulate different processing speeds
	rand.Seed(time.Now().UnixNano())
	n := rand.Intn(5) + 1
	time.Sleep(time.Duration(n) * time.Second)
	close(sb.caughtUpSig)
}

func (sb *StateBuilder) CaughtUp() <-chan struct{} {
	return sb.caughtUpSig
}

func NewStateBuilder() *StateBuilder {
	return &StateBuilder{
		caughtUpSig: make(chan struct{}),
	}
}

func main() {
	var wg sync.WaitGroup
	historyConsumer := NewStateBuilder()

	go historyConsumer.Start()

	// this goroutine does not care about the state
	wg.Add(1)
	go func() {
		fmt.Println("g1: I don't care about the state - done")
		wg.Done()
	}()

	// this goroutine needs the state to be ready
	wg.Add(1)
	go func() {
		fmt.Println("g2: need the state to be ready")
		<-historyConsumer.CaughtUp()
		fmt.Println("g2: state finally caught up")
		wg.Done()
	}()

	// this goroutine wants the state to be ready, but can do other things meanwhile
	wg.Add(1)
	go func() {
		for {
			select {
			case <-time.Tick(time.Duration(500) * time.Millisecond):
				fmt.Println("g3: still waiting")
			case <-historyConsumer.CaughtUp():
				fmt.Println("g3: state caught up")
				wg.Done()
				return
			}
		}
	}()

	wg.Wait()
	fmt.Println("Done")
}
```

This is barely scratching the surface and in a following post I'll be showing how this can
be done in a more realistic environment, such as when using Apache Kafka for the event log storage.

### But... Will it scale?

If there is the need for the service to be scaled horizontally and for all the instances to share the
state, then an in-memory store is no longer suitable but this pattern can still be useful.

The major problem to solve would be the coordination between the instances when
they need to re-create / update the store (after some downtime for example).
One way is to ensure only one instance can write to the store while others wait, or
you can have some strategy to partition writes between instances.
Signalling whether a particular instance should write or wait for the store depends greatly on
the storage system.

## Key takeaways:

When there is the need to process a backlog of messages before anything else:

- using in-memory only store can lead to lower dependency micro/nano-services
- in-memory store also allows apps to rely on in-process communication / synchronization tools
- reprocessing the entire event log on start-up, and signalling once done to unblock other threads,
can lead to a flow that is easier to reason about


[ref1]:https://martinfowler.com/eaaDev/EventSourcing.html

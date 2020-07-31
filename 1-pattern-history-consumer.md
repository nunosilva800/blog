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
For example, service health and _readyness_ checks should respond accordingly.


## Signalling when it catches up

A pattern that I've come across and grown fond of is for the thread (or group of threads) that is rebuilding
state from the log, to signal to others that depend on the state, that it is has caught up, ergo,
the state is ready.

You might already be thinking how this can be achieved with most concurrency primitives, such as locks or
semaphores, and yes, this is a good way to handle the problem.

As example of this that uses Go's channels would look like the following:

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
be done in a more realistic environment, such as when using Apache Kafka as the stream processor.

## Key takeaways:

- using in-memory only store can lead to lower dependency micro/nano-services
- reprocessing the entire event log on start up does not have to be complicated

[ref1]:https://martinfowler.com/eaaDev/EventSourcing.html

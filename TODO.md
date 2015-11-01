## Documentation
- Clarify/document at-least-once versus at-most-once decrement reliability guarantees.
- Documentation builder with TOC generator, reponame substituter?.
	- Where to keep the canonical sources of truth?
- Various inline TODOs in documentation.
- Typocheck.
- "See also" section.
- Github local-repo anchors.

#### Reliability section rough draft

Existing lock implementations rely on message delivery to the client. This causes problems if the network connection is broken between the client and RabbitMQ; without heartbeats or other AMQP operations on the connection, a client cannot easily verify that its lock is still held.

Using [amqp-locks](https://github.com/zbentley/amqp-locks) implementations, it is possible to ensure that, given a single RabbitMQ broker (not necessarily a clustered HA set; that's a [[separate can of worms]]), a lock is relinquished on the server when  (or shortly after) the server realizes that it has lost connection to the client, and will be relinquished on the client the next time it polls the availability of the lock, unless the connection is restored before the server has registered the client's absence.

Note that these assertions should be considered *speculative* so long as [amqp-locks](https://github.com/zbentley/amqp-locks) and its associated spec and proof tool are considered a work in progress. Distributed computing/locking is fraught, and until I've tested these assertions more thoroughly with [[jepsen]] or something, skepticism should be the default assumption.



## Prover
- Make prover.
	- What platform? Language doesn't matter, but ability to coordinate tests in a concurrent way does. Node is out (it DGAF when backgrounded things happen, so long as they do). Java? Clojure? Basically anything that supports the old-college-try approach to "make these two executions happen as close to simultaneously as possible".
- Jepsen? Or similar? Might just be able to repurpose the one from the rabbitmq-semaphore evisceration.

## Guidelines
- Optimized mutex with publish-then-consume on a single queue, rather than publish-then-declare-another-exclusive-queue.
- Transactions instead of publish confirms, then update doc to be generic AMQP.

## Research
- Play with old-style semaphore operations using a passive queue declare (to check consumer counts) followed by an exclusive consume to acquire a lock; one per queue.
- Play with old-style semaphore operations using a decrement lock with AUTODELETE set, such that publish-then-consume holders of the decrement lock can crash after decrementing without risking multiple decrements (in exchange for risking failed decrements; you have to choose one).
- Given a recv-based semaphore, what happens if the connection is dropped and reconnected without heartbeats?
- What happens if a consumed queue is deleted with/without heartbeats? What does the client see? Add whatever frame gets sent into the OOB examples in the parenthetical inthe "Verify problems" section.
- Are queue.declares (passive or not) more or less expensive than basic.publishes as lock-still-exists queries? If more expensive, add 'It is also marginally more expensive than the verification mechanism of [amqp-locks](https://github.com/zbentley/amqp-locks) implementations (basic.publish and confirm).' to the "Verify problems" section.
- Expirations can maybe be used to indicate to the server that a lock is no longer held?
- What happens if a connection drops, server deletes B queue, new holder acquires it, connection resumes, lock holder tries to publish to B queue? Add a UID queue? Or a consume?
- Check hard to measure assertion: if a client fills up its QoS, does it always immediately dissapear from the consumer count in queue.declare?
- Play with the LVE plugin to make last-acquired-lock data available to future semaphore acquirers to speed up acquiry in the future (this is an optimization; fallback must be available).

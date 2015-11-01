# Disclaimer/WIP/Mea Culpa
This project is ***brand new, unfinished, and a known-to-be-sketchy work-in-progress***. Don't use or take anything here as gospel; I'm still experimenting with implementations (and don't have a prover to make sure they're correct yet), so it's all subject to change. For planned research/changes, see [TODO](TODO.md).

# Project Overview

[amqp-locks](https://github.com/zbentley/amqp-locks) is a set of guidelines for implementing several types of lock primitive using [RabbitMQ](https://www.rabbitmq.com/).

**Implementations of these guidelines are available are available elsewhere (see the "Implementations" section)**; the main [amqp-locks](https://github.com/zbentley/amqp-locks) project will eventually contain three things:

- An overview of the locking behavior.
- A list of implementations of those guidelines. 
- A set of proof tools to verify that a given implementation meets the requirements, in terms of correctness and efficiency.

#### Lock Types
[amqp-locks](https://github.com/zbentley/amqp-locks) implementations should provide three (well, two and a subtype) kinds of locks. Each type is summarized below, with a link to more detailed documentation for the guidelines of implementation behavior for each type.

1. [A non-persistent mutex](Documentation/Lock Types/True Mutex.md), or "true mutex". This is a named mutex whose availability is contingent _only_ on  the availability of RabbitMQ and whether or not the lock is already held.
2. [A persistent semaphore](Documentation/Lock Types/Persistent Semaphore.md). This is a semaphore with an externally managed number of slots. Clients can hold locks, each of which consumes one slot. The number of slots can be centrally changed such that additional slots can be made available, or existing slots can be removed. If slots with held locks are removed, their lock holders will release them the next time they poll/verify their locks. Until a slot is released by a client, it cannot be claimed by another client, regardless of changes in the number of slots.
	- The bulk of [amqp-locks](https://github.com/zbentley/amqp-locks) is geared towards providing a correct implementation of this lock type; the other lock types came about as part of my research towards making a proper persistent semaphore.
3. [A persistent mutex](Documentation/Lock Types/Persistent Mutex.md). This is a named mutex whose availability is contingent on the availability of RabbitMQ, whether or not the mutex has been centrally enabled, and whether or not the lock is already held.
	- This is really just an administerable semaphore with a single slot (which allows for slightly simpler/cheaper operations than a full semaphore).

Additionally, documentation will be provided (it's still being fleshed out) for [an administerable version of the standard receive/consume-based semaphore implementation](Documentation/Lock Types/[TODO] Non-Persistent Semaphore.md).

#### Lock Requirements
Locks provided by [amqp-locks](https://github.com/zbentley/amqp-locks) implementations should be:

- As correct as possible, given the [limitations imposed by RabbitMQ itself](https://www.rabbitmq.com/partitions.html).
- Cheap to acquire. Attempting to acquire a lock should not require reconnecting to RabbitMQ. It should also try to minimize the chance of needing to close/reopen channels.
- Cheap to verify. Once a lock has been acquired by a client, it should be cheap in terms of time and RabbitMQ impact to verify that it is still held.
- Remotely administerable (where applicable). This means that it should be easy to centrally change the availability of a lock in such a way that clients, when they next attempt to acquire an unheld lock or verify a held lock, release/do not get the lock (it should not be possible during normal operations to centrally decrement and then increment a semaphore such that a new client can falsely hold it).

#### Limitation Tolerance
Locks should be able to operate in constrained situations, such as:

- Limited RabbitMQ/AMQP feature support: the drivers used in lock clients are limited in many language/platform sitautions. Multiple channels/consumers are poorly handled in some cases, and using the same channel for both RPC (like queue declaration) and consume operations is often tricky (or buggy).
	- Drivers should be able to be used for [amqp-locks](https://github.com/zbentley/amqp-locks) implementations provided they support the following:
		- basic.publish
		- publish confirmations
		- basic.queue_delete
		- basic.queue_declare and exclusive queues.
	- Drivers do not need to support consumtion (of any kind), heartbeats, or other features in order to be used for [amqp-locks](https://github.com/zbentley/amqp-locks) implementations. In the future, it may be possible to use AMQP transactions instead of publisher confirms in some cases, which would allow the use of non-RabbitMQ AMQP brokers.

- Limited concurrency support in the client: not all lock clients have a concurrency system; many are single-threaded, non-multiplexed/evented processes. This limits the usefulness or availability of AMQP heartbeats, and makes it harder to reason about locking schemes that rely on AMQP message delivery.

# Implementations

# Limitations of Existing Approaches

The excellent [semaphores in RabbitMQ](https://www.rabbitmq.com/blog/2014/02/19/distributed-semaphores-with-rabbitmq/) article proposes a way to implement a semaphore in RabbitMQ. However, there are several drawbacks to using unacked messages/message-receive operations as the core of a RabbitMQ-based locking scheme.  Using that strategy leads to locks that are:

- Hard to administer (centrally increment/decrement);
- Greedy (can steal lock "slots" in not-immediately-obvious ways);
- Hard to verify (ensure a lock is still acquired);
- Hard to measure (see how many slots are in a semaphore);
- Expensive (in terms of channel destruction/creation) to try acquire.

#### More Info
These drawbacks (and [amqp-locks](https://github.com/zbentley/amqp-locks)'s methods of coping with them) are explained in more detail in the [Existing Approaches](Documentation/Existing Approaches.md).

# Non-RabbitMQ AMQP brokers
At present, many of the locking semantics in [amqp-locks](https://github.com/zbentley/amqp-locks) are specific to RabbitMQ. Support is planned for other AMQP brokers (by using transactions instead of publisher confirms, or, if that proves too costly in time and broker operations, by using repeated exclusive queue declarations and rebuilding channels more frequently).

# Additional Resources
- [Discussion of exclusive queues and consumers](http://rabbitmq.1065348.n5.nabble.com/Exclusive-queues-and-delete-on-disconnect-td14931.html). 
- [Original (consume-based) semaphore proposal](https://www.rabbitmq.com/blog/2014/02/19/distributed-semaphores-with-rabbitmq/). 
- Aphyr's [article on reliability of RabbitMQ-backed semaphores](https://aphyr.com/posts/315-call-me-maybe-rabbitmq) (amqp-lock implementations probably do not provide reliability in excess of his criticisms).
	- Also see the twitter [discussion](https://twitter.com/aphyr/status/436610754083815425) on the same topic.
- Pythonishvili's [implementation of the original consume-based semaphore](https://github.com/pythonishvili/rabbit-semaphore/blob/master/semaphore.py).
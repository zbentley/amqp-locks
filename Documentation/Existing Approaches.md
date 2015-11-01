# Limitations of Receive-Based Locking

The excellent [semaphores in RabbitMQ](https://www.rabbitmq.com/blog/2014/02/19/distributed-semaphores-with-rabbitmq/) article proposes a way to implement a semaphore in RabbitMQ. *That semaphore is likely faster than the ones provided by [amqp-locks](https://github.com/zbentley/amqp-locks)*, as it uses AMQP consumers to hold locks, rather than RPC operations.

As this amqp-locks and its associated client implementations leave the work-in-progress stage, I will make performance benchmarks that put that assumption to the test. 

There are several drawbacks to using unacked messages/message-receive operations as the core of a RabbitMQ-based locking scheme. Those drawbacks are listed below.

### Hard to Administer

Unacked-message-based semaphore operations make it easy to add lock slots to the system; simply publish more messages into the semaphore queue. However, decrementing the number of lock slots is not as easy. To quote [the original article](https://www.rabbitmq.com/blog/2014/02/19/distributed-semaphores-with-rabbitmq/) on the subject:

  > If we wanted to decrease the number of processes that can simultaneously access a resource, then we would have to stop the running ones and purge the queue. Another way would be to start an extra consumer with very high priority so it would take as many messages as are needed from the semaphore queue and acknowledge them so they get removed from the system.

This requires consumer priorities (a recent RabbitMQ feature), or some separate management system for use in decrementing the lock. A second semaphore with one slot (i.e. a mutex) could be used to register "decrement pending", such that lock holders could attempt to acquire that mutex in order to decrement the original semaphore, but that would have several issues: it would be prone to race conditions (see next section), and would perform poorly (failure to acquire would always break the channel; see "Expensive to fail to acquire lock").

Even once a decrement operation is confirmed necessary (by receiving, but not acking, a decrement message), the process of decrementing a semaphore on the client is far from simple: the original lock channel has to be blocked by channel.flow, or have its QoS reduced so that acknowledging the already-held lock message doesn't allow another one to be queued. Then, both the decrement-request and the original lock message have to be acknowleded. Depending on the order of the acknowledgements, a failure of either one can cause false decrement success (lock is still at previous level), or false decrement failure (lock is decremented, but decrement request is still pending).

Administering [amqp-locks](https://github.com/zbentley/amqp-locks) locks, on the other hand, is trivial: a mutex (itself a [amqp-locks](https://github.com/zbentley/amqp-locks) lock) is acquired to prevent duplicate administration operations, and then lock slots are created or destroyed as queue.declare or queue.delete operations. Since those operations are both idempotent (unlike consume and publish), it is even possible to cope with situations where the broker goes down after successfully declaring/deleting a queue, but before the "declare/delete ok" message is sent to the lock administrator client.

Using [amqp-locks](https://github.com/zbentley/amqp-locks) locks, it is also possible for a lock administrator to wait until a newly-created lock slot is taken, or for it to wait until another client releases a lock that is in the process of being decremented.

### Greedy

Using receive-based locking, if a client wants a lock, it allows flow on its consuming channel (or sets QoS), and tries to receive a message. If receive is non-blocking (e.g. a `select` with a 0 timeout on the socket), a lock consumer can buffer messages in the background without explicitly acquiring the lock. Since locally buffered messages are unavailable to other lock clients, the situation of "start consuming lock queue; try to get lock (fail); do *time-consuming operation*; try to get lock (success)" results in a lock being unavailable to other clients, but unknown by the client that will eventually hold it, for the duration of *time-consuming operation*.

This can be remedied by making receive a blocking operation (which is not always desirable, especially in clients without threads/concurrency systems), or by turning flow/QoS on and off on either side of a lock-acquire attempt. That's expensive in terms of time and broker operations; in the failed-to-acquire case, two RPC actions (flow on, flow off) are necessary to poll for a message, and a timeout is still needed to robustly check for a lock, as the broker can't be counted on to deliver a message to a consumer as soon as flow is enabled.

[amqp-locks](https://github.com/zbentley/amqp-locks) implementations, on the other hand, use publishes and RPC actions without state (no local receive queue "stealing" messages) to acquire locks. While waiting on messages via receive is almost certainly faster than [amqp-locks](https://github.com/zbentley/amqp-locks)'s strategy, lock-acquire attempts with [amqp-locks](https://github.com/zbentley/amqp-locks) cannot cause hidden lock unavailability to other clients based on previous acquire attempts.

### Hard to Verify

When using unacked messages as indicators of held locks, it is difficult to verify that a lock is still held. Without heartbeats (which may not be available in many synchronous/single threaded AMQP clients, especially those which need to hold a lock for a long/highly variable time), verifying that a lock is still held requires another AMQP operation to be conducted on the original semaphore's connection to verify its aliveness. Since exclusive consumption of a queue is not idempotent, some other operation must be used. This requires driver support for multiple channels and/or frame multiplexing, since the response for the "other operation" might end up behind another frame (data or OOB message like connection.close)from the server. Such driver features may not always be present.

If a lock is decremented or centrally destroyed, there is no easy way for consume/receive/publish-based clients to check whether a their lock status (holding an acquired lock, or attempting to consume a message from an existent one) is valid.

Both the "is the connection still alive?" and "does the lock still exist?"  questions (but not the "has the lock been decremented?" question) can be answered by issuing a passive queue.declare statement for the original lock queue on the lock-holding channel. While this still requires multiplexed driver support, it does not require multichannel support.

### Hard to Measure

Using unacknowledged messages, it is hard-nee-imposible without an HTTP/management operation to determine how many slots exist on a lock when one or more slot is held by a client (as was done in [this implementation](https://github.com/pythonishvili/rabbit-semaphore) of consume-based locking). Since consumers disappear from the value returned by basic.queue_declare when their QoS/local buffer is full, the number of unacknowledged messages is not available to lock administrators.

[amqp-locks](https://github.com/zbentley/amqp-locks) implementations can check the size of a properly structured semaphore by simply issuing basic.publish operations to each slot's persistent queue in order, and recording the count at which operations started failing. This counting operation can be wrapped in a lock-administration mutex in order to prevent race conditions in which counts are externally changed by incrementing or decrementing semaphores.

### Expensive to fail to acquire lock

In a consume based lock implementation, if the desired lock does not exist at all, an attempt to consume from the lock queue will fail such that the channel must be reopened. This is expensive relative to simpler operations ([amqp-locks](https://github.com/zbentley/amqp-locks) uses a publish and a confirm) and requires multiple RPC operations.
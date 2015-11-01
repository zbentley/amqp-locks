# Persistent Semaphore

A persistent semaphore is a multi-slot lock with the number of possible available slots persistent in RabbitMQ, in which each slot can only be held by one client. It is implemented using pairs of queues; persistent `A` queues (which represent persistent semaphore "slots"), and exclusive/transient `B` queues, which represent lock holders:

## "A" and "B" Queues

A persistent semaphore is defined as a group of queues such that, given a semaphore name `N`, and a number of slots `C`, members of three sets of queues may exist in RabbitMQ:

- **`A` queues:** These are queues named some equivalent of `N-1-A`, `N-2-A`, and so on, up to `N-C-A`. These queues should persist independently of lock clients' actions.
- **`B` queues:** These are queues named some equivalent of `N-X-B`, where `C'` can be any *current or previous* value of `C`, and `0 < X <= C'`. These queues represent lock clients, and should be exclusive queues. 
-  **Administration Mutex:** This is an optional non-numbered queue named some equivalent of `N` such that its name cannot collide with the names of any `A` or `B` queues for any values of `C`. This queue represents the mutex acquired by lock administrators for lock administration actions.

`A` and `B` queue set members are represented in array-like notation (e.g. `A[2]` for `N-2-A`) in the rest of this document.

## Determining Queue Existence:
Queue existence can be determined by publishing (with confirms) to a queue or by passively declaring it.

Publishes are recommended to determine the max, as passive queue declaration's failure mode is a channel-breaking error, which would then necessitate that the lock administrator hold open two channels: one for the administration mutex and one to probe the maximum count.

Note that this does not guarantee queue identity in more than name; two publishes to queue `Q` may arrive at two different incarnations of a queue by that name.

## Determining the Maximum Number of Slots

Lock administrators can determine the maximum number of slots on a semaphore `N` by checking the existence of all `A` queues of the semaphore *in order*. When the first NO_ROUTE or NOT_FOUND occurs, the highest numbered routable queue is represents the maximum of the semaphore.


## Lock Administration


#### Creating the Semaphore
To create a semaphore, a lock administrator should create all of the `A` queues for a semaphore, from `A[1]` to `A[C]`.

#### Destroying the Semaphore
An administrator can destroy a semaphore by determining the maximum value `C` of the semaphore, and then deleting `A` queues in *descending* order from `C` to `0`.

- For alternatives to descending order, see the "Swiss Cheese Semaphores" section.

#### Adding Slots
Adding some number `X` of slots to a semaphore consists of determining the maximum value `C` of the semaphore and then declaring the `A` queue for `C + X`.

#### Removing Slots
Removing some number `X` slots from a semaphore consists of determining the maximum value `C` of the semaphore, and then deleting `A` queues in *descending* order from `C` to `max(0, C - X)`.

- Node that ***decrementing the available slots on a semaphore does not affect any lock clients that have acquired the slots being decremented.*** A client holding a decremented slot *must* voluntarily relinquish its slot (cleanly or by crashing) for the decrement operation to be complete. This is the largest limitation of REPONAME's implementation of a persistent semaphore.
- For alternatives to descending order, see the "Swiss Cheese Semaphores" section.
- It is imperative that clients wait for the delete-ok message from the broker before continuing on to remove the next queue.

#### Waiting on Lock Holders
Administrators can wait for lock holders to release a slot by polling the existence of a `B` queue for a given slot. When existence-checking fails, the lock has been relinquished. This can be done at any time, but is only really useful once the `A` queue for a slot has been removed, since a lock could be relinquished and then reacquired between successive polls in other circumstances.
	
#### Note on Linear Ordering/"Swiss Cheese Semaphores""
- Administrators can optimistically delete in *ascending* order, provided that, expiration mechanisms exist to keep old `A` queues from accumilating on a system (or overshooting is enabled such that all old `A` queues will be deleted at some point), and that lock clients are always starting from 0 when searching for an available slot on the semaphore.
- Deleting in ascending order will allow for faster lock-acquire failures in clients that start checking semaphore availability at 0, but is less reliable--if deletion fails at some number `X; 0 < X < C`, potential or current lock holders at slots between `X` and `C` will still have valid slots on a nonexistent semaphore until some "pruning" action takes place (see "overshooting").

#### Administration Mutex
Before all administrative actions, the lock administrator must acquire a mutex that is unique to the semaphore in question. This prevents multiple administrators making concurrent changes. Otherwise, it is possible to create a non-sequential set of `A` queues such that it is impossible to determine the semaphore's maximum.

#### Overshooting
Creation, addition, and deletion operations may choose to "overshoot" the determined-max value `C` of the semaphore by some amount `T`, and (idempotently, if using RabbitMQ 3 or later) delete all `A` queues found between `C` and `C + T`. This copes with some "unexpected" scenarios (multiple semaphores with different implementations/administration queues, strange broker corruption/failures, manual/external queue deletion, and so on).


### Lock Clients

#### Acquiring a Lock

Clients attempting to acquire a slot on a semaphore should do the following:

1. For some number `L` that starts at 1, poll the existence of the `A` queue for `L`.
2. If `A[L]` does not exist, return lock-acquire failure.
3. If `A[L]` exists, poll the existence of the `B` queue for `L`.
4. If `B[L]` exists, increment `L` by 1 and go to step 1.
5. If `B[L]` does not exist, attempt to declare it (all `B` queues are exclusive).
6. If declaration fails with RESOURCE_LOCKED, re-connect the channel, re-set `L` to 1 and go to step 1.
	- The whole persistent semaphore scheme is just an optimization to avoid this channel-breaking race condition
	- You can re-set `L` to any other number (or leave it at the current value) depending on lock-traffic and the likelihood of finding a free slot, but 1 usually makes the most sense.
	- It is theoretically possible for clients to infinitely recurse on lock-acquire attempts. Because of this possibility, clients should support a maximum number of failed second-stage lock acquire attempts after which they will return a failure. It may also be useful in some situations to reduce the chance of such races by introducing a single random time skew upon second-stage acquire failure before retrying.
7. If declaration succeeds, return lock-acquire success.
	- It is strongly recommended that clients also return or store the value of `L` for the lock they successfully acquired. This allows for cheap lock verification to occur.
	- It is recommended that clients verify their held lock before returning success.

Clients may use algorithms other than "start at 0 and ascend" for finding a free slot on a semaphore; depending on the lock traffic characteristics of a given application, other approaches may be more efficient. However, non-linear approaches may incur penalties in reliability and efficiency of semaphore destruction (see "Destroying the semaphore" in the "Lock administration" section). 

#### Verifying a Held Lock

Clients that have already successfully acquired a held lock should do the following to determine if the lock and/or their channel/connection are still valid:

1. Given a held lock number `L` and the connection and channel that originally acquired it, poll the existence of the `A` queue for `L`.
2. If `A[L]` does not exist, return lock-relinquished/verification-failure.
3. If `A[L]` exists, return lock-still-held/verification-success.

#### Relinquishing a Lock

Given a held lock number `L` and the connection and channel that originally acquired it, a client may do any of the following to relinquish a held semaphore:

- Close its channel.
- Close its connection (via disconnect or crash).
	- ***Important reliability note:*** If connections are closed via crash, RabbitMQ may not detect the absence of a client until its heartbeat interval, internal TCP keepalive registry update, or subsequent attempt to use the channel.
- Delete the `B` queue for `L`.


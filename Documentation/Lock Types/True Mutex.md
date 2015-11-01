# True Mutex

A true mutex is a lock reflected by a single exclusive resource. It can be implemented in one of several ways:

- As an exclusive, predictably-named queue.
	- In this implementation, verification can be separate from declaration by publishing to the mutex, provided that publishing occurs on the same channel on which the queue was declared, and that the channel is not usable for publishes if it breaks and is re-established. 
- As a declaration of a non-exclusive, auto-delete queue, and then an exclusive consumption of that queue.
	- If queue TTL will not result in the deletion of a consumed queue, a very short TTL can be used in this implementation to prevent leaving waste on the broker and to minimize the chance of wasteful channel creation when two clients attempt to acquire the mutex at the same time (if client A declares the queue and client B succeeds in 	consuming it, client A's channel will break when it attempts to consume the queue).

True mutexes do not persist as data objects outside of their use by clients.
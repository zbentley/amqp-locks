# Persistent Mutex

A persistent mutex is a mutex whose availability depends on whether it is held *and* whether or not the mutex object is available. It can be implemented in one of two ways:

- As a persistent semaphore with only one slot.
- As a persistent queue. Clients would attempt to publish to the queue (or do a passive declaration to verify its existence) to verify a mutex's existence and prevent the chance of channel breakage due to nonexistent locks, and would then exclusively consume from the queue.
	- Once a consume is established, either QoS or flow should be reduced so that other clients' polling for the existence of the lock does not fill up lock-holding clients' receive buffers.
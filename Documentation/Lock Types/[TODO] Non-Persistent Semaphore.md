This document is incomplete and will be fleshed out as I add or test implementations of non-exclusive-queue-based locking.

# [TODO] Non-Persistent Semaphore

A non-persistent semaphore is one a semaphore whose "slot" count is incremented or decremented by lock clients. Eventually, this section will contain behavior guidelines for such a semaphore. Until then, the best documentation on the subject is the [original RabbitMQ blog post on the subject](https://www.rabbitmq.com/blog/2014/02/19/distributed-semaphores-with-rabbitmq/).
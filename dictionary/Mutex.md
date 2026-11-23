---
description: A lock that allows only one thread at a time into a section of code, protecting shared data from concurrent access.
---

A mutex (mutual exclusion lock) is a synchronization primitive that lets only one [thread](./Thread.md) at a time run a protected section of code. A thread locks it before touching shared data and unlocks it afterwards. Any other thread attempting to lock it waits. The protected code is called the critical section, and a mutex is a standard fix for a [race condition](./Race%20condition.md) on in-memory state.

```go
mu.Lock()
defer mu.Unlock() // released on every return path
balance -= amount
```

Release the lock on every path, using `defer`, `finally` or a scoped guard. If an exception skips the unlock, every later caller waits forever and that code path hangs.

Locks have a performance cost when contended. Since only one thread can hold the mutex, a busy critical section turns parallel work into a single-file line. The symptom is latency that climbs with load while CPU stays low, because threads spend their time waiting.

Practical rules:

| Rule                                   | Reason                                                     |
| -------------------------------------- | ---------------------------------------------------------- |
| Keep the critical section short        | Reduces waiting                                            |
| Do not do network or disk calls inside | One slow call stalls every thread needing the lock         |
| Lock consistently                      | One unprotected read or write is still a data race         |
| Acquire several locks in a fixed order | Prevents a [deadlock](./Deadlock.md)                       |
| Do not re-lock unless it is reentrant  | Go's `sync.Mutex` deadlocks if the holder locks it again   |

A read-write lock lets many readers in together but gives a writer exclusive access. A reentrant lock, such as Java's `synchronized`, can be taken again by the thread already holding it.

A mutex works only inside one process. It does nothing for two server instances, so rules across instances belong in the database or another shared coordinator. In single-threaded [async-await](./Async-await.md) code, an async mutex can protect a sequence that spans an `await`.

_Usage:_

"Latency goes up with traffic, but the CPU's only at 20%."

"Check the mutex around the cache. It's held during a network call, so every request queues behind it."

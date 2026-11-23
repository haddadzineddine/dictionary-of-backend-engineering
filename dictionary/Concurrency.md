---
description: Structuring a program so several tasks are in progress over the same period, whether or not they run at the same instant.
---

Concurrency is structuring a program so that several tasks are in progress during the same period. The tasks may take turns on a single CPU core, which distinguishes it from [parallelism](./Parallelism.md), where tasks execute at the same instant. A server needs concurrency because requests overlap: while one waits for a database response, another can use the CPU.

Languages and runtimes offer different models:

| Model                                  | Who switches tasks                  | Examples                          |
| -------------------------------------- | ----------------------------------- | --------------------------------- |
| OS [threads](./Thread.md)              | Operating system, at any moment     | Java platform threads, C++        |
| Runtime-managed threads                | Language runtime                    | Goroutines, Java virtual threads  |
| [Event loop](./Event%20loop.md) tasks  | Tasks yield at [await](./Async-await.md) points | Node.js, Python asyncio |
| Multiple processes                     | Operating system, memory isolated   | Pre-fork web servers              |

Concurrency helps most when work spends its time waiting, such as on network calls, disk access or a database. It does little for a task that uses the CPU continuously, because there is no idle time for another task to use.

The difficulty comes from shared state. When tasks overlap, their steps can interleave in orders you did not plan for. That produces the typical concurrency bug: code passes every test with one user, then under load a counter comes up short or two customers book the same seat. Such a [race condition](./Race%20condition.md) may appear only rarely, which makes it hard to reproduce.

Protecting shared data with a [mutex](./Mutex.md) solves that but adds waiting, and careless locking can produce a [deadlock](./Deadlock.md). Where possible, avoid sharing mutable state, or let the database enforce the rule atomically.

Limit concurrency as well. Launching one task per item in a list of 100,000 can exhaust connections, memory or a partner's rate limit. A [thread pool](./Thread%20pool.md) or semaphore (a counter capping simultaneous tasks) keeps the number bounded.

_Usage:_

"It works perfectly when I test it, but production totals are a few cents off."

"That sounds like a concurrency bug. Two requests are probably updating the same balance at once."

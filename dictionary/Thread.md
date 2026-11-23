---
description: An independent path of execution inside a process, with its own stack but sharing the process's memory with other threads.
---

A thread is an independent path of execution inside a process. Each thread has its own stack and instruction position, while all threads in a process share the same heap memory and open files. Sharing makes communication between them easy and is also the source of most [concurrency](./Concurrency.md) bugs.

The operating system's scheduler decides which threads run and can pause one at nearly any instruction. On a multi-core machine, several run at once, giving [parallelism](./Parallelism.md). Even a single line such as `count += 1` is usually several machine steps, so another thread can run between them and cause a [race condition](./Race%20condition.md).

| Unit             | Scheduled by     | Memory                | Relative cost |
| ---------------- | ---------------- | --------------------- | ------------- |
| Process          | Operating system | Isolated              | Highest       |
| OS thread        | Operating system | Shared within process | Medium        |
| Virtual thread or goroutine | Language runtime | Shared, small growable stack | Low |

The traditional server model gives each request its own thread. It is easy to reason about, because code runs top to bottom and [blocking I/O](./Blocking%20IO.md) only pauses that one thread. Its limit is cost. Each OS thread reserves stack space, and switching between many of them consumes CPU time. A service creating one thread per connection can fail during a traffic spike with an error such as Java's `unable to create native thread`, even though the CPU is barely busy.

Servers therefore reuse a fixed set through a [thread pool](./Thread%20pool.md), or use lighter units such as goroutines and Java virtual threads.

Threads in a process share its failures. An uncaught fatal error or memory corruption in one can bring down the entire process. When a service hangs, a thread dump, a snapshot of what each thread is doing, is usually the first diagnostic to take.

_Usage:_

"The service stopped responding, but CPU is at 2%."

"Take a thread dump. If every thread is parked on the same call, we've found what's holding them up."

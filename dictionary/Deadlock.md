---
description: A state where tasks wait on each other in a cycle, so none can proceed and the affected work hangs indefinitely.
---

A deadlock is a state in which two or more tasks each wait for something another holds, so none of them can ever proceed. The typical symptom is a service that stops responding with low CPU and no error in the logs, because nothing has failed.

The simplest case involves two locks:

| Step | Task A           | Task B           |
| ---- | ---------------- | ---------------- |
| 1    | Locks `accounts` |                  |
| 2    |                  | Locks `orders`   |
| 3    | Waits for `orders` |                |
| 4    |                  | Waits for `accounts` |

A deadlock requires four conditions at once, known as the Coffman conditions, and removing any one prevents it:

| Condition        | Meaning                                  | How to break it                          |
| ---------------- | ---------------------------------------- | ---------------------------------------- |
| Mutual exclusion | A resource has one holder at a time      | Use lock-free or immutable data          |
| Hold and wait    | A task holds one resource, requests more | Acquire everything at once               |
| No preemption    | A resource cannot be taken away          | Use lock timeouts and back off           |
| Circular wait    | Tasks wait in a cycle                    | Always acquire in one global order       |

Consistent ordering is the usual fix. If every code path takes a [mutex](./Mutex.md) on the lower account ID first, a transfer from A to B and one from B to A cannot form a cycle.

Deadlocks also occur without explicit locks. A [thread pool](./Thread%20pool.md) deadlocks when every [thread](./Thread.md) in it waits for subtasks queued in that same pool. A request holding one database connection while waiting for a second can deadlock a small connection pool under load. Calling `.Result` on a task in some C# contexts blocks the thread that the task needs in order to finish.

To diagnose one, take a thread dump and look for tasks waiting on each other. Relational databases detect lock cycles and abort one transaction, so applications should retry that error. Timeouts turn an indefinite hang into a visible failure.

_Usage:_

"Transfers hang now and then, always between the same two accounts."

"Opposite transfers lock in opposite order, so it's a deadlock. Always lock the lower account ID first."

---
description: A fixed set of reusable threads that take tasks from a queue, bounding how much work a server runs at once.
---

A [thread](./Thread.md) pool is a set of long-lived threads that take tasks from a queue and run them. Reusing threads avoids the cost of creating one per task, and the fixed size caps how much work runs at the same time.

| Step | Who         | What happens                                              |
| ---- | ----------- | --------------------------------------------------------- |
| 1    | Caller      | Submits a task                                            |
| 2    | Pool        | Assigns it to an idle thread, or places it in the queue   |
| 3    | Thread      | Runs the task to completion                               |
| 4    | Thread      | Returns to the pool and takes the next task               |
| 5    | Pool        | Rejects the task if the queue is full                     |

The characteristic failure is pool exhaustion. One slow dependency makes every thread wait on it, new requests pile up in the queue, and the entire service appears hung while CPU sits near idle. Endpoints that do not touch the slow dependency also time out, because they cannot get a thread. Set timeouts on every outbound call, and give risky dependencies their own small pool.

Sizing depends on the work. For CPU-bound tasks, start near the number of cores. For tasks that mostly wait on I/O, a larger pool makes sense, but keep it consistent with downstream limits: 200 threads sharing 20 database connections will queue for connections instead.

Bound the queue. An unbounded queue hides overload until memory runs out, and requests wait so long that clients have already given up. Rejecting early with a 503 [status code](./Status%20code.md) is usually better.

A task that submits a subtask to the same pool and waits for it can cause a [deadlock](./Deadlock.md) when all threads are doing this. Because threads are reused, thread-local data left by one request, such as a user ID, can leak into the next unless it is cleared.

_Usage:_

"Even the health check is timing out, and the CPU is idle."

"The thread pool's exhausted. Every thread is stuck on that payment call because it has no timeout."

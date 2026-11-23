---
description: Executing multiple computations at the same instant on separate CPU cores to finish CPU-heavy work sooner.
---

Parallelism is executing multiple computations at the same instant on separate CPU cores. It is often confused with [concurrency](./Concurrency.md), which is about managing overlapping tasks. A single-core machine can be concurrent by switching between tasks, but it cannot be parallel.

| Situation                         | Concurrent | Parallel | What helps                     |
| --------------------------------- | ---------- | -------- | ------------------------------ |
| 1,000 requests waiting on network | Yes        | Not needed | [Async-await](./Async-await.md) or many [threads](./Thread.md) |
| Resizing images on 8 cores        | Possibly   | Yes      | One task per core              |
| One long sequential calculation   | No         | No       | A faster algorithm             |

Parallelism speeds up CPU-bound work, meaning work limited by computation rather than waiting. Adding cores to an I/O-bound service often changes little, because its time is spent waiting on something else.

Speedup has a limit. Amdahl's law states that the serial portion of a program caps the benefit of adding processors. If 10% of a job must run sequentially, it can never go more than ten times faster, however many cores you add. Coordination, such as waiting on a [mutex](./Mutex.md), adds further cost, and past some point more threads can make a program slower.

Runtimes differ in what they allow. Node.js runs JavaScript on one main thread, so CPU-heavy work needs `worker_threads` or separate processes. CPython's standard build has a global interpreter lock that stops threads running Python code simultaneously, so CPU-bound Python typically uses multiple processes. Optional free-threaded builds have been available since Python 3.13. Go and Java run threads across cores by default.

Containers add a common surprise. A runtime that sizes itself from the host's core count may start far more threads than its CPU limit allows, and the container gets throttled, showing latency spikes while average CPU looks moderate. Check that pool sizes reflect the container's limit.

_Usage:_

"We doubled the cores and the API didn't get any faster."

"It's I/O-bound. Parallelism won't help when every request is waiting on the same database."

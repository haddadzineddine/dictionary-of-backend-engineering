---
description: A long-running process that takes jobs from a queue and executes them, separately from the processes serving web requests.
---

A worker is a long-running process that repeatedly takes a [background job](./Background%20job.md) from a [job queue](./Job%20queue.md), runs it and reports the result. Workers usually share the application's codebase but run as separate processes from the web servers, so heavy jobs do not take capacity from user requests and each side can scale independently.

A worker has its own [concurrency](./Concurrency.md) setting, the number of jobs it runs at once. The right value depends on the work:

| Job type            | Suggested setup                                     | Reason                             |
| ------------------- | --------------------------------------------------- | ---------------------------------- |
| CPU-heavy           | About one job per core                              | More only adds switching           |
| I/O-heavy           | Higher concurrency                                  | Jobs mostly wait                   |
| Memory-heavy        | Low concurrency, dedicated workers                  | Prevents out-of-memory kills       |
| Rate-limited API    | A limit shared across workers                       | Per-process limits add up          |

Adding workers is not always safe. Each one opens connections, so doubling the count can exhaust the database's connection limit and turn a slow backlog into errors for the web application too. Check downstream capacity before scaling, and scale on queue age or depth instead of CPU when jobs mostly wait.

Prefetching causes another surprise. Some systems let a worker reserve several jobs in advance. A worker holding a batch while stuck on one long job leaves the others waiting even though other workers are idle. Lower the prefetch setting for long-running jobs.

Deploys need care because old and new code run at the same time. A new web server can enqueue a job in a format an old worker cannot read, so deploy changes to job payloads in compatible steps. Workers also need [graceful shutdown](./Graceful%20shutdown.md): on a stop signal, stop taking new jobs and finish or release current ones.

_Usage:_

"We added ten workers to clear the backlog and now the API is throwing connection errors."

"Each worker holds its own connections, so we've hit the database limit. Scale back and lower their concurrency."

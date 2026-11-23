---
description: A durable list of pending background jobs that producers add to and workers take from, with retries and failure handling.
---

A job queue is a store of pending [background jobs](./Background%20job.md). Application code adds jobs, and [workers](./Worker.md) take them and run them. The queue separates when work is requested from when it is done, so traffic bursts build up as a backlog instead of overloading the system. Sidekiq, BullMQ and Celery are examples.

A job moves through a lifecycle:

| Step | Who    | What happens                                                   |
| ---- | ------ | -------------------------------------------------------------- |
| 1    | App    | Enqueues a job with its type and arguments                     |
| 2    | Worker | Reserves it, making it invisible to others for a lease period  |
| 3    | Worker | Runs the job                                                   |
| 4    | Worker | Acknowledges success, and the queue removes it                 |
| 5    | Queue  | On failure or expired lease, makes it available for retry      |
| 6    | Queue  | After too many attempts, moves it to a dead-letter queue       |

Most queues provide at-least-once delivery. If a worker finishes a job and crashes before acknowledging it, the job runs again. A lease that is shorter than the job's run time causes the same effect while the first worker is still going: two workers process one job and customers receive duplicate emails. Set the lease above the expected duration or extend it while running, and make jobs idempotent.

Retries need delays. Immediate retries against a struggling dependency add load, so use exponential backoff (doubling the wait) with random jitter. Cap the attempts so a poison job, one that can never succeed, does not loop forever. The dead-letter queue holds such jobs for inspection and needs an alert, or failures accumulate unseen.

Monitor the age of the oldest job in addition to queue length. A thousand jobs may be fine for one queue, while a five-minute delay on password-reset emails is an incident. Separate queues by urgency so a large bulk import cannot delay time-sensitive work.

_Usage:_

"Password reset emails are arriving forty minutes late."

"The marketing send filled the job queue. Put transactional email on its own queue with dedicated workers."

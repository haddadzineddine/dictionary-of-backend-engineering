---
description: A task that runs automatically on a time-based schedule, such as every night at 2 a.m.
---

A cron job is a task run automatically on a recurring schedule. The name comes from `cron`, the Unix scheduler, whose five-field expression is now used by many other schedulers. Nightly reports, cleanup of expired data and monthly invoicing are typical uses.

```text
0 2 * * 1-5   run at 02:00, Monday to Friday
```

| Field        | Allowed values |
| ------------ | -------------- |
| Minute       | 0-59           |
| Hour         | 0-23           |
| Day of month | 1-31           |
| Month        | 1-12           |
| Day of week  | 0-7 (0 and 7 are Sunday) |

If both day fields are restricted, standard cron runs when either matches, so `0 0 13 * 5` fires on every 13th and on every Friday.

Most cron problems come from assumptions about how a run behaves:

| Problem           | Symptom                                       | What to do                                   |
| ----------------- | --------------------------------------------- | -------------------------------------------- |
| Runs per instance | Scaling to three servers sends three invoices | Use one scheduler, or take a shared lock     |
| Overlapping runs  | A slow job is still going when the next starts | Skip the run if the last is still active    |
| Daylight saving   | A 02:30 job is skipped or runs twice          | Schedule in UTC                              |
| Missed runs       | The host was down at 02:00, so nothing ran    | Process "everything due" instead of "today"  |
| Silent failure    | Backups stopped weeks ago and nobody noticed  | Alert when an expected success does not arrive |

The last row needs a specific control. An alert on failure cannot fire for a job that never started, so have each successful run send a heartbeat and alert when one is missing.

Write the job so that running it twice is harmless, for example by recording which billing period has been processed. Keep the cron entry small: for large tasks, have it enqueue a [background job](./Background%20job.md) on a [job queue](./Job%20queue.md), which provides retries and visibility.

_Usage:_

"Customers got three copies of the monthly invoice."

"The cron job runs on every instance, and we scaled to three. Move it to a single scheduler and make it idempotent."

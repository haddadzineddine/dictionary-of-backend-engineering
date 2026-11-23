---
description: Work moved outside the request-response path so it can run later, retry on failure and not keep the user waiting.
---

A background job is a unit of work that runs outside the [request lifecycle](./Request%20lifecycle.md). The handler records that the work must happen, responds to the user and leaves execution to another process. Sending email, generating reports, resizing images and syncing with third parties are typical cases.

The symptom that calls for one is a request doing too much. Signup takes eight seconds because it calls an email provider, and when that provider has an outage, signups start failing even though the account could have been created. Moving the email to a job keeps the endpoint fast and isolates the failure.

Jobs are usually placed on a [job queue](./Job%20queue.md) and executed by a [worker](./Worker.md). If the client needs the outcome, the API can return `202 Accepted` with a job ID to check, or notify it through a [webhook](./Webhook.md) or [Server-Sent Events](./Server-Sent%20Events.md).

Job code needs different habits from request code:

| Rule                             | Reason                                                  |
| -------------------------------- | ------------------------------------------------------- |
| Make it idempotent               | Most queues can deliver a job more than once            |
| Pass IDs, not whole objects      | Data may have changed by the time the job runs          |
| Set a timeout                    | A hung job holds a worker indefinitely                  |
| Keep the payload compatible      | Old jobs may run after new code deploys                 |
| Recheck permissions and state    | The user may have been removed or the order cancelled   |

Timing against the database transaction causes a frequent bug. If you enqueue before the commit, a worker may start immediately, fail to find the row and report "record not found". If you commit and then crash before enqueueing, the job is lost. A transactional outbox solves both: write the job to a table in the same transaction as the data, and have a separate process publish it.

_Usage:_

"Checkout is slow because we generate the PDF receipt in the handler."

"Make it a background job. Return the order now and email the receipt when it's ready."

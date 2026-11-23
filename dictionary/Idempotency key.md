---
description: A unique client-generated value sent with a request so that retries repeat the original result instead of the side effect.
---

An idempotency key is a unique value a client attaches to a request so the server performs the operation at most once, however many times the request is retried. It addresses a specific problem: a customer clicks Pay, the connection times out, the client retries, and the customer is charged twice. After a timeout, the client cannot know whether the first attempt succeeded.

A [POST](./HTTP%20method.md) is not naturally idempotent, so the key supplies that property. The client generates a value such as a UUID for each logical operation and reuses it on every retry, commonly in an `Idempotency-Key` header.

| Step | Who    | What happens                                                               |
| ---- | ------ | -------------------------------------------------------------------------- |
| 1    | Client | Generates a key and sends the request                                      |
| 2    | Server | Atomically inserts the key with a hash of the request, marked in progress  |
| 3    | Server | If the key is new, runs the operation                                      |
| 4    | Server | Stores the response with the key                                           |
| 5    | Client | Retries with the same key after a failure                                  |
| 6    | Server | Returns the stored response without repeating the work                     |

The insert must rely on a unique constraint, because a separate "check, then insert" lets two concurrent retries both pass, which is a [race condition](./Race%20condition.md). A request arriving during the first should get a 409 conflict. If the same key arrives with a different body, reject it.

Scope keys to the account or API client so customers cannot collide. Keep them for longer than your longest retry window, then expire them.

Where possible, commit the key and the business change in one database transaction. If the operation calls an external provider, pass a key to that provider too, otherwise a crash between its success and your commit can still repeat the charge. A [webhook](./Webhook.md) receiver needs similar deduplication, using the event ID.

_Usage:_

"A customer got charged twice after the app timed out."

"The retry created a second payment. Generate one idempotency key per checkout and reuse it on retries."

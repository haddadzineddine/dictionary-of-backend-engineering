---
description: An HTTP request one system sends to a URL you register, notifying you that an event happened.
---

A webhook is an [HTTP](./HTTP.md) request that one system sends to a URL you registered, to tell you an event occurred. A payment provider might POST to `/webhooks/payments` when a charge succeeds. It replaces polling the provider for changes.

Your endpoint is public and deliveries are unreliable, so follow a fixed sequence:

| Step | What to do                                                           | Why                                  |
| ---- | -------------------------------------------------------------------- | ------------------------------------ |
| 1    | Read the raw request body                                            | Signatures cover the exact bytes     |
| 2    | Verify the signature using a constant-time comparison                | Anyone can POST to a public URL      |
| 3    | Reject stale timestamps if the provider signs one                    | Limits replay of captured requests   |
| 4    | Store the event, ignoring an ID already seen                         | Deliveries can repeat                |
| 5    | Enqueue a [background job](./Background%20job.md) and return 2xx     | Slow handlers trigger retries        |

Most providers deliver at least once and retry on any non-2xx response or timeout. If your handler does slow work before responding, the provider times out and resends, and the customer receives two confirmation emails. Acknowledge quickly and process afterwards.

Signature failures are a common integration bug. Many providers sign the raw body with HMAC, a keyed hash, and a JSON parser in your [middleware](./Middleware.md) that re-serializes the body changes the bytes, so valid requests fail verification. Capture the raw body for this route. Do not respond by disabling the check.

Delivery order is generally not guaranteed. A `subscription.updated` event can arrive before `subscription.created`. Use timestamps or version numbers in the payload, or fetch the current state from the provider's API when an event arrives.

If your product sends webhooks to customer-supplied URLs, that feature is an [SSRF](./SSRF.md) risk. Validate destinations and block internal addresses. Sign each delivery, retry with increasing delays and let customers replay failures.

_Usage:_

"The provider says the webhook failed, but our handler logged success."

"The handler takes 40 seconds, so they timed out and retried. Queue the work and return 200 right after verifying."

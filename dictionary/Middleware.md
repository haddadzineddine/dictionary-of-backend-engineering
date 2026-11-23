---
description: Code that wraps request handlers to apply shared behaviour such as logging, authentication and rate limiting, in a set order.
---

Middleware is code that runs around a request handler and applies behaviour shared by many routes. Logging, [authentication](./Authentication.md), compression, rate limiting and error formatting are common examples. It occupies one stage of the [request lifecycle](./Request%20lifecycle.md), so individual handlers do not each repeat the same logic.

Middleware forms a chain. Each function receives the request, may do some work, then either calls the next function or ends the request itself. When the inner function returns, control passes back outward, so the same middleware can act on both the request and the response:

```text
logging -> auth -> rate limit -> handler
logging <- auth <- rate limit <- handler
```

Order changes behaviour, and most middleware bugs are ordering bugs:

| Ordering mistake                       | What you observe                                        |
| -------------------------------------- | ------------------------------------------------------- |
| Logging placed after auth              | Rejected requests never appear in the logs              |
| Error handler placed too far in        | Some exceptions return a default HTML 500 page          |
| Per-user rate limit before auth        | No user is known yet, so the limit cannot apply per user |
| Body parser before a signature check   | A [webhook](./Webhook.md) signature fails because the raw bytes are gone |
| Auth registered after a route          | That route is reachable with no login                   |

The last row is a security failure that raises no error. In frameworks where registration order matters, a route defined before the auth middleware skips it. Prefer applying auth globally and marking the few public routes explicitly, so a new route is protected by default.

Keep middleware for cross-cutting concerns. Business rules placed here are hard to find and run on routes that do not need them. Slow middleware also adds its cost to every request that passes through it.

_Usage:_

"Why is the admin export reachable without logging in?"

"That route is registered above the auth middleware, so it never runs. Move it below, or make auth global and allowlist public routes."

---
description: The ordered stages a request passes through on a server, from accepting the connection to sending the response and cleaning up.
---

The request lifecycle is the sequence of stages an [HTTP](./HTTP.md) request passes through inside a server. Knowing the sequence tells you where to add a check and where to look when a request is slow or fails.

A typical lifecycle in a web framework:

| Step | Who                 | What happens                                                        |
| ---- | ------------------- | ------------------------------------------------------------------- |
| 1    | Proxy or server     | Accepts the connection and terminates [TLS](./TLS.md) if configured |
| 2    | Server              | Parses the request line, headers and, when needed, the body         |
| 3    | Router              | Matches the method and path to a handler                            |
| 4    | [Middleware](./Middleware.md) | Runs logging, [authentication](./Authentication.md), rate limits |
| 5    | Handler             | Validates input and calls application code                          |
| 6    | Application code    | Reads and writes data, calls other services                         |
| 7    | Handler             | Builds the response and [status code](./Status%20code.md)           |
| 8    | Middleware          | Runs in reverse for response headers, logging and error mapping     |
| 9    | Server              | Writes the response and keeps or closes the connection              |

When a request is slow, measure each stage instead of guessing. Time spent before the handler starts often means requests are queued, for example waiting for a free slot in a [thread pool](./Thread%20pool.md) or for a busy [event loop](./Event%20loop.md). That produces a confusing symptom: the handler's own timing looks fast while clients see long delays.

A client disconnect does not automatically stop the work. If the user closes the tab or a proxy times out, many frameworks keep running the handler, so a request reported as failed may still commit its database write. Pass a cancellation signal or deadline down through your calls where the framework supports it.

Work that does not need to finish before the response belongs in a [background job](./Background%20job.md), which keeps this path short.

_Usage:_

"The trace says the handler took 40 ms, but the client waited three seconds."

"Then the time is earlier in the request lifecycle. Check whether requests are queueing before the handler starts."

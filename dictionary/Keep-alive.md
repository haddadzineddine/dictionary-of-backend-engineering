---
description: Reusing one open connection for several HTTP requests, avoiding repeated setup. Mismatched idle timeouts cause intermittent 502s.
---

Keep-alive is the practice of leaving a connection open after a response so the next request can reuse it. Opening a new connection costs a TCP handshake plus, on encrypted connections, a [TLS](./TLS.md) handshake. Reuse removes those round trips from every request after the first. In [HTTP](./HTTP.md)/1.1, connections persist by default unless one side sends `Connection: close`, and HTTP/2 is built around one long-lived connection.

Either side may close an idle connection, and this creates a well-known failure. Suppose a proxy or load balancer (a server that spreads traffic across instances) keeps idle connections to your app for 60 seconds, while your app closes them after 5. The proxy can send a new request down a connection at the moment the app is closing it. The result is a small, irregular number of 502 errors or `ECONNRESET` messages, often with nothing in the application logs because the request never reached a handler.

The usual fix is to keep each server's idle timeout longer than its client's:

| Hop               | Idle timeout rule                               |
| ----------------- | ----------------------------------------------- |
| Proxy to app      | App must keep connections open longer than the proxy |
| App to dependency | Client pool should retire connections before the server closes them |

The same applies to outbound calls. An HTTP client with a connection pool can pick up a connection the remote server has already closed, so the request fails immediately. Good clients detect this and retry when it is safe, which depends on the [HTTP method](./HTTP%20method.md).

Keep-alive here is different from TCP keepalive, an operating-system probe that checks whether an idle peer is reachable. Long-lived connections also complicate [graceful shutdown](./Graceful%20shutdown.md), since a server must close idle connections before it can exit.

_Usage:_

"We get a handful of 502s an hour and the app logs show nothing."

"Sounds like keep-alive. The app's idle timeout is shorter than the load balancer's, so raise it above the balancer's value."

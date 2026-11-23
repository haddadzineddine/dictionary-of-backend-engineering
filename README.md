# Backend Engineering Dictionary

A plain-English dictionary of building server applications: HTTP, API design, authentication, concurrency, background work, code structure and testing.

Why did the payment run twice? Why does the test pass alone and fail in CI? Why does one slow call freeze the whole server?

Each of these has a short explanation once you know the right words. Entries define the term, describe the failure it explains where there is one, and end with a short example of engineers using it.

---

## Table of contents

- **[Section 1 — Requests and Protocols](#section-1--requests-and-protocols)**
  - [HTTP](#http)
  - [HTTP method](#http-method)
  - [Status code](#status-code)
  - [Request lifecycle](#request-lifecycle)
  - [Middleware](#middleware)
  - [Keep-alive](#keep-alive)
  - [TLS](#tls)
  - [WebSocket](#websocket)
  - [Server-Sent Events](#server-sent-events)
  - [Long polling](#long-polling)
- **[Section 2 — API Design](#section-2--api-design)**
  - [REST](#rest)
  - [GraphQL](#graphql)
  - [gRPC](#grpc)
  - [Protocol Buffers](#protocol-buffers)
  - [OpenAPI](#openapi)
  - [Pagination](#pagination)
  - [Cursor pagination](#cursor-pagination)
  - [API versioning](#api-versioning)
  - [Breaking change](#breaking-change)
  - [Idempotency key](#idempotency-key)
  - [Webhook](#webhook)
- **[Section 3 — Identity and Security](#section-3--identity-and-security)**
  - [Authentication](#authentication)
  - [Authorization](#authorization)
  - [Session cookie](#session-cookie)
  - [JWT](#jwt)
  - [OAuth 2.0](#oauth-20)
  - [OpenID Connect](#openid-connect)
  - [RBAC](#rbac)
  - [ABAC](#abac)
  - [API key](#api-key)
  - [Password hashing](#password-hashing)
  - [CSRF](#csrf)
  - [SQL injection](#sql-injection)
  - [SSRF](#ssrf)
- **[Section 4 — Concurrency](#section-4--concurrency)**
  - [Concurrency](#concurrency)
  - [Parallelism](#parallelism)
  - [Thread](#thread)
  - [Thread pool](#thread-pool)
  - [Event loop](#event-loop)
  - [Blocking IO](#blocking-io)
  - [Async-await](#async-await)
  - [Race condition](#race-condition)
  - [Mutex](#mutex)
  - [Deadlock](#deadlock)
- **[Section 5 — Background Work](#section-5--background-work)**
  - [Background job](#background-job)
  - [Job queue](#job-queue)
  - [Worker](#worker)
  - [Cron job](#cron-job)
  - [Graceful shutdown](#graceful-shutdown)
- **[Section 6 — Code Structure](#section-6--code-structure)**
  - [Layered architecture](#layered-architecture)
  - [Hexagonal architecture](#hexagonal-architecture)
  - [Dependency injection](#dependency-injection)
  - [Repository pattern](#repository-pattern)
  - [DTO](#dto)
  - [ORM](#orm)
  - [Domain-driven design](#domain-driven-design)
  - [Bounded context](#bounded-context)
- **[Section 7 — Testing](#section-7--testing)**
  - [Unit test](#unit-test)
  - [Integration test](#integration-test)
  - [Contract test](#contract-test)
  - [Test double](#test-double)
  - [Test pyramid](#test-pyramid)
  - [Flaky test](#flaky-test)

## Section 1 — Requests and Protocols

### HTTP

HTTP (Hypertext Transfer Protocol) is the protocol a client and server use to exchange messages. The client sends a request containing an [HTTP method](#http-method), a path, headers and an optional body. The server returns a response containing a [status code](#status-code), headers and an optional body.

HTTP is stateless, which means the protocol keeps no memory of earlier requests. Each request must carry whatever the server needs to process it. Logged-in behaviour is built on top, usually by sending a [session cookie](#session-cookie) or a [JWT](#jwt) every time. This explains a common surprise: a call that works in the browser returns 401 when copied into a script, because the script did not send the credentials the browser was attaching.

The protocol has three versions in common use:

| Version  | Transport     | How requests share a connection                                 |
| -------- | ------------- | --------------------------------------------------------------- |
| HTTP/1.1 | TCP           | One response at a time, in order, so clients open several       |
| HTTP/2   | TCP           | Many concurrent streams over one connection                     |
| HTTP/3   | QUIC over UDP | Many streams, and a lost packet delays only its own stream      |

Application code rarely changes between versions, because handlers see the same methods, headers and bodies. The differences show up in performance. On HTTP/2, one lost TCP packet delays every stream on that connection until it is retransmitted, which is the problem HTTP/3 was designed to remove.

Connections are normally encrypted with [TLS](#tls) and reused through [keep-alive](#keep-alive). HTTP defines the message format but not the design of your API. Conventions such as [REST](#rest) sit on top of it, and [gRPC](#grpc) uses HTTP/2 as its transport.

_Usage:_

"The endpoint works in Chrome but curl gets a 401."

"HTTP is stateless, so curl knows nothing about your login. Copy the cookie header across and try again."

### HTTP method

An [HTTP](#http) method is the verb at the start of a request. It tells the server what kind of operation the client intends, and it tells every system in between, such as caches and proxies, how that request may be treated.

Two properties matter. A safe method does not ask the server to change anything. An idempotent method can be sent several times with the same intended effect as sending it once.

| Method  | Typical use             | Safe | Idempotent |
| ------- | ----------------------- | ---- | ---------- |
| GET     | Read a resource         | Yes  | Yes        |
| HEAD    | Read headers only       | Yes  | Yes        |
| OPTIONS | Ask what is supported   | Yes  | Yes        |
| PUT     | Replace a resource      | No   | Yes        |
| DELETE  | Remove a resource       | No   | Yes        |
| POST    | Create or trigger       | No   | No         |
| PATCH   | Partially update        | No   | Not by default |

These properties are promises your handler has to keep, because the protocol does not enforce them. A `GET /unsubscribe?id=42` link that changes data looks harmless until a link previewer, crawler or email scanner fetches it and unsubscribes the user with no click. State changes belong behind a non-safe method, which also supports [CSRF](#csrf) defences.

Idempotency decides what can be retried automatically. Clients and proxies may resend a GET or PUT after a dropped connection. Retrying a POST can create a second order or a second charge, because the client cannot tell whether the first attempt succeeded before the connection died. An [idempotency key](#idempotency-key) makes such a POST safe to retry.

DELETE stays idempotent even if the second call returns 404, since idempotency describes the state of the server and not the response.

_Usage:_

"Half our trial users got unsubscribed and none of them clicked anything."

"The unsubscribe link is a GET, so the mail scanner triggered it. Make the link open a page and do the change with a POST."

### Status code

A status code is the three-digit number at the start of an [HTTP](#http) response. It tells the client the outcome in a form software can act on, and the first digit gives the class.

| Class | Meaning      | Common examples                                                  |
| ----- | ------------ | ---------------------------------------------------------------- |
| 1xx   | Informational | 101 Switching Protocols                                         |
| 2xx   | Success      | 200 OK, 201 Created, 202 Accepted, 204 No Content                |
| 3xx   | Redirection  | 301 Moved Permanently, 304 Not Modified, 307 Temporary Redirect  |
| 4xx   | Client error | 400, 401, 403, 404, 409 Conflict, 422, 429 Too Many Requests     |
| 5xx   | Server error | 500, 502 Bad Gateway, 503 Unavailable, 504 Gateway Timeout       |

The class drives automated behaviour. Clients commonly retry 503 and other temporary failures, and they should not retry most 4xx responses, because the same request will fail the same way. Monitoring usually alerts on the 5xx rate. An API that returns `200 OK` with `{"error": "failed"}` in the body hides failures from all of this: dashboards stay green while users see errors.

Some codes are often confused. 401 means the request lacks valid credentials, so it is an [authentication](#authentication) failure despite being named "Unauthorized". 403 means the server knows who you are and refuses, which is an [authorization](#authorization) failure. Some APIs return 404 instead of 403 so they do not reveal that a private resource exists.

The 5xx codes also point to where a failure happened. 500 usually comes from your application. 502 and 504 usually come from a proxy in front of it, reporting a bad response or no response from the application. If logs show no error for a 502, check the proxy and the connection between it and the app, including [keep-alive](#keep-alive) timeouts.

_Usage:_

"Error rate is flat but support says checkout is broken."

"Checkout returns a 200 with an error in the body, so the alert never fires. Return a proper status code."

### Request lifecycle

The request lifecycle is the sequence of stages an [HTTP](#http) request passes through inside a server. Knowing the sequence tells you where to add a check and where to look when a request is slow or fails.

A typical lifecycle in a web framework:

| Step | Who                 | What happens                                                        |
| ---- | ------------------- | ------------------------------------------------------------------- |
| 1    | Proxy or server     | Accepts the connection and terminates [TLS](#tls) if configured |
| 2    | Server              | Parses the request line, headers and, when needed, the body         |
| 3    | Router              | Matches the method and path to a handler                            |
| 4    | [Middleware](#middleware) | Runs logging, [authentication](#authentication), rate limits |
| 5    | Handler             | Validates input and calls application code                          |
| 6    | Application code    | Reads and writes data, calls other services                         |
| 7    | Handler             | Builds the response and [status code](#status-code)           |
| 8    | Middleware          | Runs in reverse for response headers, logging and error mapping     |
| 9    | Server              | Writes the response and keeps or closes the connection              |

When a request is slow, measure each stage instead of guessing. Time spent before the handler starts often means requests are queued, for example waiting for a free slot in a [thread pool](#thread-pool) or for a busy [event loop](#event-loop). That produces a confusing symptom: the handler's own timing looks fast while clients see long delays.

A client disconnect does not automatically stop the work. If the user closes the tab or a proxy times out, many frameworks keep running the handler, so a request reported as failed may still commit its database write. Pass a cancellation signal or deadline down through your calls where the framework supports it.

Work that does not need to finish before the response belongs in a [background job](#background-job), which keeps this path short.

_Usage:_

"The trace says the handler took 40 ms, but the client waited three seconds."

"Then the time is earlier in the request lifecycle. Check whether requests are queueing before the handler starts."

### Middleware

Middleware is code that runs around a request handler and applies behaviour shared by many routes. Logging, [authentication](#authentication), compression, rate limiting and error formatting are common examples. It occupies one stage of the [request lifecycle](#request-lifecycle), so individual handlers do not each repeat the same logic.

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
| Body parser before a signature check   | A [webhook](#webhook) signature fails because the raw bytes are gone |
| Auth registered after a route          | That route is reachable with no login                   |

The last row is a security failure that raises no error. In frameworks where registration order matters, a route defined before the auth middleware skips it. Prefer applying auth globally and marking the few public routes explicitly, so a new route is protected by default.

Keep middleware for cross-cutting concerns. Business rules placed here are hard to find and run on routes that do not need them. Slow middleware also adds its cost to every request that passes through it.

_Usage:_

"Why is the admin export reachable without logging in?"

"That route is registered above the auth middleware, so it never runs. Move it below, or make auth global and allowlist public routes."

### Keep-alive

Keep-alive is the practice of leaving a connection open after a response so the next request can reuse it. Opening a new connection costs a TCP handshake plus, on encrypted connections, a [TLS](#tls) handshake. Reuse removes those round trips from every request after the first. In [HTTP](#http)/1.1, connections persist by default unless one side sends `Connection: close`, and HTTP/2 is built around one long-lived connection.

Either side may close an idle connection, and this creates a well-known failure. Suppose a proxy or load balancer (a server that spreads traffic across instances) keeps idle connections to your app for 60 seconds, while your app closes them after 5. The proxy can send a new request down a connection at the moment the app is closing it. The result is a small, irregular number of 502 errors or `ECONNRESET` messages, often with nothing in the application logs because the request never reached a handler.

The usual fix is to keep each server's idle timeout longer than its client's:

| Hop               | Idle timeout rule                               |
| ----------------- | ----------------------------------------------- |
| Proxy to app      | App must keep connections open longer than the proxy |
| App to dependency | Client pool should retire connections before the server closes them |

The same applies to outbound calls. An HTTP client with a connection pool can pick up a connection the remote server has already closed, so the request fails immediately. Good clients detect this and retry when it is safe, which depends on the [HTTP method](#http-method).

Keep-alive here is different from TCP keepalive, an operating-system probe that checks whether an idle peer is reachable. Long-lived connections also complicate [graceful shutdown](#graceful-shutdown), since a server must close idle connections before it can exit.

_Usage:_

"We get a handful of 502s an hour and the app logs show nothing."

"Sounds like keep-alive. The app's idle timeout is shorter than the load balancer's, so raise it above the balancer's value."

### TLS

TLS (Transport Layer Security) is the protocol that encrypts a connection and lets the client verify which server it is talking to. [HTTP](#http) sent over TLS is HTTPS. It provides confidentiality, so observers cannot read the traffic; integrity, so changes are detected; and server [authentication](#authentication) through a certificate.

A connection begins with a handshake:

| Step | Who    | What happens                                                         |
| ---- | ------ | -------------------------------------------------------------------- |
| 1    | Client | Sends supported versions, cipher suites and key-agreement data       |
| 2    | Server | Chooses parameters and sends its certificate chain                   |
| 3    | Client | Checks the chain, expiry date and that the hostname matches          |
| 4    | Both   | Derive shared session keys                                           |
| 5    | Both   | Encrypt application data with those keys                             |

TLS 1.3 completes this in one round trip, and TLS 1.2 takes two. Use TLS 1.2 or later, since 1.0 and 1.1 are deprecated. [Keep-alive](#keep-alive) avoids repeating the handshake on every request.

Most TLS incidents are certificate problems. An expired certificate takes a service down for every client at once, so automate renewal and alert on the expiry date. A server that omits its intermediate certificate can work in a browser, which may already have it cached, and fail with `unable to verify` from a server-side client. A hostname mismatch fails when a service is called by an address the certificate does not list.

Do not fix these errors by disabling verification, such as `verify=False` or `NODE_TLS_REJECT_UNAUTHORIZED=0`. The traffic stays encrypted, but the client will accept an attacker's certificate. Install the correct CA certificate (the issuer's root of trust) instead.

TLS protects data only in transit and only between its two endpoints. If a proxy terminates TLS, the hop behind it is plaintext unless it is encrypted separately.

_Usage:_

"The Python client throws a certificate error, but the site is fine in Chrome."

"The server's probably missing its intermediate cert. Fix the chain and leave TLS verification on."

### WebSocket

A WebSocket is a persistent connection over which client and server can each send messages at any time. It suits features where both sides talk frequently, such as chat, collaborative editing and multiplayer state.

The connection starts as an ordinary [HTTP](#http) request carrying an `Upgrade: websocket` header. The server answers with [status code](#status-code) `101 Switching Protocols`, and from then on the same connection carries WebSocket frames instead of HTTP messages. Use the encrypted `wss://` scheme, which runs over [TLS](#tls).

Long-lived connections change how a server behaves in production:

| Problem                  | Symptom                                     | What to do                                       |
| ------------------------ | ------------------------------------------- | ------------------------------------------------ |
| Idle timeout at a proxy  | Sockets drop after a fixed quiet period     | Send ping/pong heartbeats                        |
| Half-open connections    | Server keeps "online" users who have left   | Close sockets that miss heartbeats               |
| Deploys                  | Every client disconnects at once            | Reconnect with random backoff, drain slowly      |
| Several server instances | A message misses users on another instance  | Publish through a shared message broker          |
| Slow consumers           | Server memory grows                         | Limit buffers and disconnect slow clients        |

Clients must assume the connection will drop, and must be able to catch up on anything missed after reconnecting. Without random delay, a deploy causes all clients to reconnect at the same instant.

Security needs specific attention. Browsers attach cookies to the handshake, and WebSockets are not restricted by the same-origin policy, so a malicious page can open a socket as the logged-in user. Validate the `Origin` header during the handshake, perform [authentication](#authentication) there, and check [authorization](#authorization) on each message.

If data flows only from server to client, [Server-Sent Events](#server-sent-events) are simpler to run.

_Usage:_

"Everyone's live cursor freezes after about a minute of inactivity."

"The proxy is closing idle WebSocket connections. Add a heartbeat every 30 seconds and reconnect on close."

### Server-Sent Events

Server-Sent Events (SSE) is a standard for streaming messages from a server to a client over one long-running [HTTP](#http) response. The server replies with `Content-Type: text/event-stream`, leaves the response open and writes events as they happen:

```text
id: 42
event: order-updated
data: {"orderId": 7, "status": "shipped"}

```

A blank line ends each event. In browsers, the built-in `EventSource` API parses the stream and reconnects automatically after a drop. On reconnect it sends the last ID it received in a `Last-Event-ID` header, so the server can replay what was missed if it has kept those events.

SSE fits notifications, progress bars, live dashboards and streamed text. Because it is plain HTTP, it works with existing [middleware](#middleware), cookies and routing. Client-to-server messages travel as normal requests. When both directions are frequent, a [WebSocket](#websocket) is a better fit.

The most common problem is buffering. A proxy or compression layer may hold output until it has a full chunk, so events that were sent one at a time arrive in a late batch, or only when the stream ends. Disable buffering and compression for this route at each hop. With nginx, for example, the response header `X-Accel-Buffering: no` turns off proxy buffering.

Other limits to plan for:

| Limit                 | Effect                                                     | Response                              |
| --------------------- | ---------------------------------------------------------- | ------------------------------------- |
| HTTP/1.1 browser cap  | About six connections per origin, shared across tabs       | Serve over HTTP/2                     |
| Idle timeouts         | Quiet streams get closed                                   | Send a comment line such as `: ping`  |
| No custom headers     | `EventSource` cannot set `Authorization`                   | Use a cookie or a fetch-based client  |
| Open connections      | Each client holds server resources                         | Prefer an [event loop](#event-loop) server |

_Usage:_

"The progress bar sits at zero, then jumps to 100 at the end."

"The events are being buffered by nginx. Turn buffering off for the SSE route."

### Long polling

Long polling is a technique for receiving updates over ordinary [HTTP](#http) requests. The client sends a request, and the server holds it open instead of answering immediately. When new data appears, the server responds. If nothing happens before a timeout, it returns an empty response such as `204 No Content`. Either way, the client immediately sends the next request.

Regular polling asks on a fixed schedule, so most responses are empty and updates wait for the next tick. Long polling delivers an update soon after it occurs and needs no special protocol, so it works through most proxies and firewalls.

| Technique | Direction | Latency | Connection cost | Infrastructure |
| --------- | --------- | ------- | --------------- | -------------- |
| Short polling | Client asks | Up to one interval | Repeated requests | Plain HTTP |
| Long polling | Server to client | Low | One held request per client | Plain HTTP |
| [Server-Sent Events](#server-sent-events) | Server to client | Low | One open stream | Plain HTTP, no buffering |
| [WebSocket](#websocket) | Both | Low | One open socket | Upgrade support |

A common bug is losing events between polls. If an event fires after one response is sent and before the next request arrives, a server that only reports "what happens while you are connected" never delivers it. Have the client send the ID or cursor of the last event it received, and return everything after that point.

Set the hold time below every timeout in the path. If a proxy gives up at 30 seconds and the server holds for 60, clients see regular 504 errors during quiet periods. Add a short random delay after errors so a restart does not bring every client back at once.

Held requests also consume capacity. On a server that dedicates a [thread](#thread) to each request, a few hundred waiting clients can exhaust the pool, so this pattern fits non-blocking servers better.

_Usage:_

"Users sometimes miss a notification until they refresh."

"Our long polling drops anything that fires between requests. Send the last event ID on each poll."

## Section 2 — API Design

### REST

REST (Representational State Transfer) is an architectural style for networked APIs, described by Roy Fielding in his 2000 doctoral dissertation. A REST API exposes resources, such as orders or users, at URLs. Clients act on them through standard [HTTP methods](#http-method) and exchange representations of them, usually JSON.

| Operation           | Request               | Typical success                                   |
| ------------------- | --------------------- | ------------------------------------------------- |
| List orders         | `GET /orders`         | 200                                               |
| Read one order      | `GET /orders/42`      | 200                                               |
| Create an order     | `POST /orders`        | 201 with a `Location` header                      |
| Replace an order    | `PUT /orders/42`      | 200 or 204                                        |
| Update some fields  | `PATCH /orders/42`    | 200 or 204                                        |
| Delete an order     | `DELETE /orders/42`   | 204                                               |

Fielding's constraints include a client-server split, stateless requests, cacheable responses, a uniform interface and a layered system. Statelessness means each request carries its own context, so any server instance can handle it. Cacheability means GET responses can be reused by browsers and proxies when their headers allow it.

Most APIs described as REST follow only part of this. The full style includes hypermedia, where responses contain links telling the client what it can do next, and few JSON APIs implement it. In everyday use, "REST" usually means resource-oriented JSON over [HTTP](#http).

The practical benefit comes from following the conventions, because clients, proxies and monitoring already understand them. Endpoints such as `POST /getUser` or a `200` response wrapping an error discard that shared understanding: responses cannot be cached, retries are unsafe to automate and error rates read as zero. Return an accurate [status code](#status-code).

REST can be awkward when clients need differently shaped data, leading to several round trips or oversized responses. [GraphQL](#graphql) addresses that case, and [gRPC](#grpc) is common for internal service calls. An [OpenAPI](#openapi) document is the usual way to describe a REST API.

_Usage:_

"Should cancelling an order be a DELETE?"

"No, the order still exists afterwards. In REST terms, either PATCH its status or POST to a `/orders/42/cancellation` resource."

### GraphQL

GraphQL is a query language and server runtime for APIs. The server publishes a typed schema describing its data, and each client sends a query naming the fields it wants, usually to a single endpoint over [HTTP](#http). The response has the same shape as the query.

```graphql
query {
  order(id: "42") {
    status
    customer { name }
  }
}
```

This addresses two problems common with [REST](#rest): fetching more data than a screen needs, and making several round trips to assemble one view. The server implements a resolver, a small function that fetches one field.

That flexibility moves work and risk to the server:

| Problem            | Symptom                                               | What to do                                   |
| ------------------ | ----------------------------------------------------- | -------------------------------------------- |
| N+1 queries        | A list of 50 items triggers 51 database queries       | Batch lookups per request, as DataLoader does |
| Expensive queries  | A deeply nested query ties up the server              | Enforce depth and cost limits, plus timeouts |
| Field-level access | A private field is reachable through another type     | Apply [authorization](#authorization) in the business layer |
| HTTP caching       | Queries sent by POST bypass shared caches             | Use persisted queries sent by GET            |
| Error handling     | Failures hidden behind a 200                          | Monitor the `errors` array in responses      |

The access-control row deserves care. Because the graph can reach the same object along several paths, a check placed on one resolver can be bypassed by another path to the same data. Put the rule in code that every path shares.

Public GraphQL APIs accept arbitrary queries from strangers, so limits are required. For APIs used only by your own clients, an allowlist of persisted queries, where only pre-registered queries can run, closes off most of this risk.

Evolving a schema usually means adding fields and marking old ones `@deprecated` instead of using [API versioning](#api-versioning). Removing a field is still a [breaking change](#breaking-change).

_Usage:_

"The orders page got slow after we added the customer's name."

"That's N+1 in the GraphQL resolver. It fetches each customer separately, so batch them with a loader."

### gRPC

gRPC is a framework for calling a function on another service as if it were a local method. You define services and messages in a [Protocol Buffers](#protocol-buffers) file, then generate client and server code for your languages. Calls run over [HTTP](#http)/2 and their payloads are compact binary.

```proto
service Orders {
  rpc GetOrder (GetOrderRequest) returns (Order);
}
```

gRPC supports four call shapes:

| Type                    | Client sends | Server sends | Example                  |
| ----------------------- | ------------ | ------------ | ------------------------ |
| Unary                   | One message  | One message  | Fetch an order           |
| Server streaming        | One message  | Many         | Stream search results    |
| Client streaming        | Many         | One message  | Upload metric batches    |
| Bidirectional streaming | Many         | Many         | Live two-way sync        |

It is most common between internal services, where shared schemas and generated clients prevent mismatched payloads. Browsers cannot speak native gRPC because their APIs do not expose the required HTTP/2 control, so browser clients need gRPC-Web or a translating proxy. For public APIs, [REST](#rest) remains easier for outside developers to consume.

Deadlines are a standard feature and should be set on every call. A deadline says how long the caller will wait, and it propagates to downstream gRPC calls when the context is passed along. A service with no deadline can hang indefinitely on a stuck dependency, tying up its own capacity while it waits.

A frequent production surprise concerns load balancing. Because HTTP/2 multiplexes many calls over one long-lived connection, a balancer working at the connection level sends all of a client's calls to one instance. New instances then sit idle while one is overloaded. Use a balancer that understands individual requests, or client-side balancing.

gRPC reports errors with its own codes, such as `UNAVAILABLE` and `DEADLINE_EXCEEDED`, which are separate from HTTP [status codes](#status-code).

_Usage:_

"We scaled to six pods but one is still taking all the traffic."

"gRPC keeps one HTTP/2 connection open, so the L4 balancer pins it to one pod. We need request-level balancing."

### Protocol Buffers

Protocol Buffers, or Protobuf, is a schema language and binary serialization format created at Google. You define messages in a `.proto` file, and a compiler generates types for each language you use. It is the default message format for [gRPC](#grpc).

```proto
message Order {
  string id = 1;
  int64 total_cents = 2;
  optional string coupon = 3;
}
```

The numbers are field numbers, and they are what goes on the wire. Field names are not sent in the binary format, which keeps messages small. It also means the number is the identity of the field: two programs agree on what a value means only because they agree on its number.

This drives the rules for changing a schema safely:

| Change                     | Safe? | Why                                                         |
| -------------------------- | ----- | ----------------------------------------------------------- |
| Add a field with a new number | Yes | Old readers skip fields they do not know                    |
| Remove a field and reserve it | Yes, if readers tolerate its absence | `reserved` stops the number being reused |
| Rename a field             | Binary yes, JSON no | The number is unchanged, but JSON uses names and generated code changes |
| Change a field's type      | Usually no | Only some types are compatible, and values may be truncated |
| Reuse an old field number  | No    | Old data and old clients misread the new field              |

Reusing a number is the failure to watch for. Nothing errors. A service still running the old schema decodes the new value as the old field, producing wrong data that can go unnoticed. When deleting a field, add `reserved 3;` so the compiler rejects reuse. Treat any unsafe change as a [breaking change](#breaking-change).

In proto3, a plain scalar field cannot distinguish "unset" from its default, such as `0` or an empty string. A discount of zero and a missing discount look identical. Mark the field `optional` when that difference matters.

_Usage:_

"Old consumers are suddenly showing refund amounts as customer IDs."

"Someone reused field number 4 in the Protobuf schema. Give it a new number and mark 4 as reserved."

### OpenAPI

OpenAPI is a specification for describing an [HTTP](#http) API in a YAML or JSON file. The document lists paths, [HTTP methods](#http-method), parameters, request and response bodies, [status codes](#status-code) and security schemes. It was formerly called Swagger, and that name survives in tools such as Swagger UI.

```yaml
paths:
  /orders/{id}:
    get:
      parameters:
        - { name: id, in: path, required: true, schema: { type: string } }
      responses:
        "200":
          description: The order
```

Because the file is machine-readable, one description can drive several tools: interactive documentation, generated client SDKs, mock servers, request validation and automated checks. It is most often used with [REST](#rest) style APIs.

Teams usually choose one of two workflows:

| Workflow   | Source of truth | Benefit                                | Risk                                 |
| ---------- | --------------- | -------------------------------------- | ------------------------------------ |
| Spec-first | The OpenAPI file | API is reviewed before code is written | Code can diverge without enforcement |
| Code-first | Annotated code  | Spec is generated, so less to maintain | Design follows implementation details |

The main failure is drift, where the document stops matching the running service. A consumer generates a client from the spec, ships it, then gets a validation error because a field documented as a number is actually returned as a string. Hand-maintained documents drift unless something checks them.

Reduce drift by generating the spec from code, or by validating real requests and responses against it in tests. A [contract test](#contract-test) can use the document as the agreed interface. Comparing the spec between commits in CI, the automated build pipeline, can also flag a [breaking change](#breaking-change) before it merges.

An OpenAPI file describes the shape of an API, and it says little about behaviour. It will not capture ordering guarantees, rate-limit strategy or which fields depend on each other unless you write that in its descriptions.

_Usage:_

"The generated client crashes because `amount` comes back as a string."

"So the OpenAPI spec has drifted from the service. Add response validation to the API tests so this fails in CI."

### Pagination

Pagination is splitting a large list into smaller responses that a client requests one at a time. Without it, an endpoint such as `GET /orders` works in development with 20 rows, then times out or runs the server out of memory once a customer has several hundred thousand. The server has to load, serialize and send everything in one response.

There are three common approaches:

| Approach                                    | Request                  | Strength                      | Weakness                               |
| ------------------------------------------- | ------------------------ | ----------------------------- | -------------------------------------- |
| Offset                                      | `?limit=50&offset=100`   | Simple, can jump to any page  | Slows with depth, shifts during writes |
| Page number                                 | `?page=3&per_page=50`    | Maps to numbered page UI      | Same as offset                         |
| [Cursor](#cursor-pagination)          | `?limit=50&after=abc123` | Stable and fast at any depth  | No jumping to an arbitrary page        |

Offset pagination has two known problems. The database generally has to walk past all skipped rows, so page 10,000 is much slower than page 1. Results also shift when data changes between requests. If a row is inserted at the top while a client is reading, the last item of page 1 reappears as the first item of page 2. If a row is deleted, an item gets skipped.

Some rules apply to any approach. Always enforce a server-side maximum page size, because a client-supplied `limit=1000000` removes the protection. Sort by a deterministic order that ends in a unique column, such as `created_at, id`. Without a tie-breaker, rows sharing a timestamp can appear on two pages or on none.

Be careful with total counts. A `total` field looks cheap, but counting a large filtered set can cost more than fetching the page. Consider omitting it, approximating it or returning only a `has_more` flag. Changing the pagination style of a published endpoint is usually a [breaking change](#breaking-change).

_Usage:_

"The export job keeps seeing the same invoice twice."

"It's using offset pagination while new invoices come in. Switch it to a cursor on `created_at` and `id`."

### Cursor pagination

Cursor [pagination](#pagination) is a method where each request says "give me the items after this one" instead of "skip N items". The server returns a page and a cursor, an opaque token that marks the position of the last item. The client sends that token back to get the next page.

```sql
SELECT id, created_at, total
FROM orders
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 51;
```

The cursor here encodes the `created_at` and `id` of the last row. Some databases need this row comparison written out. With an index on `(created_at, id)`, the database jumps to the position directly, so the thousandth page costs about the same as the first. Inserts and deletes elsewhere do not shift the position, which removes the duplicated and skipped rows seen with offsets.

| Step | Who    | What happens                                                    |
| ---- | ------ | --------------------------------------------------------------- |
| 1    | Client | Requests the first page with a limit of 50                      |
| 2    | Server | Fetches 51 rows, so the extra row shows whether more exist      |
| 3    | Server | Returns 50 rows and a cursor built from the 50th                |
| 4    | Client | Sends the cursor to fetch the next page                         |
| 5    | Server | Decodes and validates it, then continues from that position     |

The sort must end in a unique column. If you sort by `created_at` alone, rows with the same timestamp at a page boundary get lost.

Keep cursors opaque, for example base64-encoded, so clients do not depend on their contents and you can change the format later. Encoding is not protection, so treat a decoded cursor as untrusted input: validate it, use bound parameters to prevent [SQL injection](#sql-injection), and apply normal [authorization](#authorization) filters on every page.

The trade-off is navigation. You cannot jump to page 37, and a cursor is tied to its sort order, so changing the sort requires starting again.

_Usage:_

"Deep pages on the activity feed take seconds to load."

"That's the offset scanning past every row. Cursor pagination on `(created_at, id)` will keep it constant."

### API versioning

API versioning is a way of publishing changed behaviour without breaking clients that depend on the old behaviour. Clients state which version they expect, and the server honours that contract. It exists because you cannot update every consumer at the moment you deploy, particularly mobile apps and third-party integrations.

Versioning is needed for a [breaking change](#breaking-change). Additive changes, such as a new optional field, normally ship without a new version.

| Strategy          | Example                              | Trade-off                                       |
| ----------------- | ------------------------------------ | ----------------------------------------------- |
| URL path          | `/v2/orders`                         | Obvious and easy to route, but coarse           |
| Custom header     | `Api-Version: 2`                     | Clean URLs, harder to test in a browser         |
| Media type        | `Accept: application/vnd.acme.v2+json` | Precise, less familiar to clients             |
| Query parameter   | `/orders?version=2`                  | Easy to use, can complicate caching             |
| Dated versions    | `Api-Version: 2026-01-15`            | Fine-grained, requires a translation layer      |

If a version is selected by a header, any cache in front of the API must vary on that header. Otherwise one client can receive a response cached for another version.

The ongoing cost is maintenance. Each live version has to be tested, patched and kept secure, and copying handlers per version multiplies every future bug fix. A common structure keeps one current implementation and adds thin adapters that translate older requests and responses at the boundary.

A version also needs a retirement plan. Track usage per version and per client, announce a date, and signal it in responses, for example with the standard `Sunset` header. Contact remaining users before the date. Teams that skip this step end up supporting `/v1` indefinitely because nobody knows who still calls it.

[GraphQL](#graphql) APIs tend to avoid numbered versions by deprecating fields, and [Protocol Buffers](#protocol-buffers) schemas evolve through compatible changes, but both still need a plan for removing old fields.

_Usage:_

"Can we finally delete `/v1`?"

"Not yet. API versioning metrics show one partner still calling it, so send them the sunset date first."

### Breaking change

A breaking change is a modification to an interface that causes a previously correct client to fail or behave incorrectly. The symptom usually appears as a deploy that passed all of the service's own tests, followed by errors in a different team's application.

Some breaks are obvious and others are not:

| Change                                     | Breaking? | Reason                                        |
| ------------------------------------------ | --------- | --------------------------------------------- |
| Add an optional request field              | No        | Existing requests remain valid                |
| Add a response field                       | Usually no | Strict parsers may reject unknown fields     |
| Remove or rename a field                   | Yes       | Clients still read the old name               |
| Change a type, such as number to string    | Yes       | Parsing or arithmetic fails                   |
| Make an optional request field required    | Yes       | Existing requests are rejected                |
| Tighten validation                         | Yes       | Previously accepted input now fails           |
| Add an enum value                          | Often     | Exhaustive client code hits an unknown case   |
| Change a default, sort order or [status code](#status-code) | Often | Clients may rely on it      |

The last two rows catch people out. Clients depend on observed behaviour as well as on documentation, so an undocumented sort order can become something a consumer relies on. Whether a change breaks anyone depends on actual usage.

The safe way to make an incompatible change is expand and contract:

| Step | Action                                           |
| ---- | ------------------------------------------------ |
| 1    | Add the new field or endpoint beside the old one |
| 2    | Serve both and migrate consumers                 |
| 3    | Measure until the old one has no traffic         |
| 4    | Remove the old one                               |

When this is impractical, publish the change under [API versioning](#api-versioning). Catch breaks before release by diffing an [OpenAPI](#openapi) or [Protocol Buffers](#protocol-buffers) schema in CI, the automated build pipeline, and by running a [contract test](#contract-test) for each known consumer. Note that during a rolling deploy, old and new versions run at once, so even internal changes must be compatible across that window.

_Usage:_

"We only renamed `userId` to `user_id`. Why is mobile crashing?"

"That's a breaking change. Old app versions still read `userId`, so return both until they've gone."

### Idempotency key

An idempotency key is a unique value a client attaches to a request so the server performs the operation at most once, however many times the request is retried. It addresses a specific problem: a customer clicks Pay, the connection times out, the client retries, and the customer is charged twice. After a timeout, the client cannot know whether the first attempt succeeded.

A [POST](#http-method) is not naturally idempotent, so the key supplies that property. The client generates a value such as a UUID for each logical operation and reuses it on every retry, commonly in an `Idempotency-Key` header.

| Step | Who    | What happens                                                               |
| ---- | ------ | -------------------------------------------------------------------------- |
| 1    | Client | Generates a key and sends the request                                      |
| 2    | Server | Atomically inserts the key with a hash of the request, marked in progress  |
| 3    | Server | If the key is new, runs the operation                                      |
| 4    | Server | Stores the response with the key                                           |
| 5    | Client | Retries with the same key after a failure                                  |
| 6    | Server | Returns the stored response without repeating the work                     |

The insert must rely on a unique constraint, because a separate "check, then insert" lets two concurrent retries both pass, which is a [race condition](#race-condition). A request arriving during the first should get a 409 conflict. If the same key arrives with a different body, reject it.

Scope keys to the account or API client so customers cannot collide. Keep them for longer than your longest retry window, then expire them.

Where possible, commit the key and the business change in one database transaction. If the operation calls an external provider, pass a key to that provider too, otherwise a crash between its success and your commit can still repeat the charge. A [webhook](#webhook) receiver needs similar deduplication, using the event ID.

_Usage:_

"A customer got charged twice after the app timed out."

"The retry created a second payment. Generate one idempotency key per checkout and reuse it on retries."

### Webhook

A webhook is an [HTTP](#http) request that one system sends to a URL you registered, to tell you an event occurred. A payment provider might POST to `/webhooks/payments` when a charge succeeds. It replaces polling the provider for changes.

Your endpoint is public and deliveries are unreliable, so follow a fixed sequence:

| Step | What to do                                                           | Why                                  |
| ---- | -------------------------------------------------------------------- | ------------------------------------ |
| 1    | Read the raw request body                                            | Signatures cover the exact bytes     |
| 2    | Verify the signature using a constant-time comparison                | Anyone can POST to a public URL      |
| 3    | Reject stale timestamps if the provider signs one                    | Limits replay of captured requests   |
| 4    | Store the event, ignoring an ID already seen                         | Deliveries can repeat                |
| 5    | Enqueue a [background job](#background-job) and return 2xx     | Slow handlers trigger retries        |

Most providers deliver at least once and retry on any non-2xx response or timeout. If your handler does slow work before responding, the provider times out and resends, and the customer receives two confirmation emails. Acknowledge quickly and process afterwards.

Signature failures are a common integration bug. Many providers sign the raw body with HMAC, a keyed hash, and a JSON parser in your [middleware](#middleware) that re-serializes the body changes the bytes, so valid requests fail verification. Capture the raw body for this route. Do not respond by disabling the check.

Delivery order is generally not guaranteed. A `subscription.updated` event can arrive before `subscription.created`. Use timestamps or version numbers in the payload, or fetch the current state from the provider's API when an event arrives.

If your product sends webhooks to customer-supplied URLs, that feature is an [SSRF](#ssrf) risk. Validate destinations and block internal addresses. Sign each delivery, retry with increasing delays and let customers replay failures.

_Usage:_

"The provider says the webhook failed, but our handler logged success."

"The handler takes 40 seconds, so they timed out and retried. Queue the work and return 200 right after verifying."

## Section 3 — Identity and Security

### Authentication

Authentication is the process of verifying who is making a request. It is separate from [authorization](#authorization), which decides what that identity may do. A failed authentication normally returns [status code](#status-code) 401, and a known user who lacks permission gets 403.

Authentication usually happens twice. First, a user proves their identity at login. The server then issues a credential that proves it on later requests, because [HTTP](#http) keeps no state between them.

| Credential                              | Typical caller           | Revocation                          |
| --------------------------------------- | ------------------------ | ----------------------------------- |
| [Session cookie](#session-cookie) | Browser users            | Immediate, by deleting the session  |
| [JWT](#jwt) access token            | APIs and services        | Usually waits for expiry            |
| [API key](#api-key)               | Scripts and integrations | Immediate, by deleting the key      |
| Client certificate                      | Internal services        | Through certificate revocation or short lifetimes |

Login endpoints need specific defences. Store passwords only with [password hashing](#password-hashing). Rate-limit attempts per account and per source, because attackers replay leaked credentials from other sites. Offer multi-factor authentication. Passkeys and hardware keys resist phishing, authenticator codes are weaker, and SMS is weaker still.

Avoid revealing which accounts exist. A form that says "no account with that email" lets an attacker test a list of addresses. Return the same message for a wrong email and a wrong password, and apply the same approach to password reset and signup flows. Response time can leak the same fact, so hash a dummy value when the account does not exist.

Issue a new session identifier at login, invalidate sessions on logout and password change, and ask for the password again before sensitive actions such as changing an email address. Where possible, use a maintained library or an identity provider through [OpenID Connect](#openid-connect) instead of writing this yourself.

_Usage:_

"Our login page says 'unknown email' when the address isn't registered."

"That leaks which accounts exist. Authentication errors should read the same for a bad email and a bad password."

### Authorization

Authorization is the decision about whether a known caller may perform a specific action on a specific resource. It runs after [authentication](#authentication), which establishes who the caller is. When the check fails, the server normally returns [status code](#status-code) 403, or 404 if the existence of the resource should stay hidden.

The most common failure is a missing check on an individual object. A logged-in user opens `/invoices/1001`, changes the number to `1002` and sees another customer's invoice. Nothing verified that this invoice belongs to this user. This flaw is known as an insecure direct object reference (IDOR), or broken object-level authorization. Unguessable IDs make it harder to exploit, but they do not fix it.

Authorization happens at several levels:

| Level    | Question                                   | Example                              |
| -------- | ------------------------------------------ | ------------------------------------ |
| Function | May this caller use this operation?        | Only admins can issue refunds        |
| Object   | May they act on this particular record?    | Users read only their own invoices   |
| Field    | May they read or set this property?        | Users cannot set their own `role`    |

Enforce every check on the server, because hiding a button changes nothing for someone calling the API directly. Deny by default, so a forgotten rule fails closed. Check on every request, including list, export and bulk endpoints, and on any [background job](#background-job) acting for a user. Scope queries to the tenant or owner, such as `WHERE id = ? AND account_id = ?`.

Keep rules in one shared place and not scattered across handlers. [RBAC](#rbac) grants permissions through roles, while [ABAC](#abac) evaluates attributes of the user, resource and context. Write tests asserting that one user cannot reach another user's data, because normal feature tests only exercise the permitted path.

_Usage:_

"QA changed the order ID in the URL and got someone else's receipt."

"That's a missing authorization check. Load the order scoped to the current account and return 404 if it isn't theirs."

### Session cookie

A session cookie is a browser cookie containing a random identifier that maps to login state on the server. After [authentication](#authentication), the server creates a session record and sends its ID with a `Set-Cookie` header. The browser attaches the cookie to later requests, and the server looks up the session to identify the user.

Because the state stays on the server, revocation is immediate: delete the record and the cookie stops working. A self-contained [JWT](#jwt) does not offer this by default. The cost is a lookup on each request and a store shared by all server instances. If sessions live in one process's memory, users appear to be logged out at random once a second instance is added or the app restarts.

Cookie attributes control exposure:

| Attribute         | Effect                                      | Recommended                          |
| ----------------- | ------------------------------------------- | ------------------------------------ |
| `HttpOnly`        | JavaScript cannot read the cookie           | Always                               |
| `Secure`          | Sent only over HTTPS                        | Always in production                 |
| `SameSite`        | Limits sending on cross-site requests       | `Lax` or `Strict`                    |
| `Path` / `Domain` | Limits where the cookie is sent             | Omit `Domain` to stay host-only      |
| `Max-Age`         | Sets a lifetime                             | As short as the product allows       |

`HttpOnly` reduces the damage of cross-site scripting (attacker JavaScript running on your page), since the script cannot steal the ID, though it can still act within the page. `SameSite` helps against [CSRF](#csrf), but state-changing routes should still have a dedicated defence.

Generate a long ID with a cryptographically secure random generator. Issue a new ID at login and on privilege changes, which prevents session fixation, where an attacker plants a known ID before the victim logs in. On logout, delete the record on the server as well as clearing the cookie. Apply both an idle timeout and an absolute expiry.

_Usage:_

"Since we added a second instance, people keep getting logged out."

"Session cookies point to in-process memory, so the other instance can't find them. Move sessions to a shared store."

### JWT

A JWT (JSON Web Token) is a compact token containing JSON claims and a signature. It has three base64url-encoded parts separated by dots: `header.payload.signature`. A server holding the right key can verify the signature and trust the claims without a database lookup, which makes JWTs common for API [authentication](#authentication) and between services.

```json
{ "iss": "https://auth.example.com", "aud": "orders-api", "sub": "user_42", "exp": 1798765432 }
```

A standard signed JWT is encoded and not encrypted. Anyone holding the token can read its payload, so never put secrets or sensitive personal data in it. The signature only proves that the contents have not changed since a trusted key signed them.

Verification must check all of the following, preferably using a maintained library:

| Check           | Failure it prevents                                                       |
| --------------- | ------------------------------------------------------------------------- |
| Signature       | Forged or edited tokens                                                   |
| Allowed `alg`   | `none` tokens and confusion between public-key and shared-secret algorithms |
| `exp` and `nbf` | Expired or not-yet-valid tokens                                           |
| `iss`           | Tokens from another issuer                                                |
| `aud`           | A token issued for one service being replayed against another             |

Configure the accepted algorithm on the server and never take it from the token's header. Decoding is not verifying: many libraries have a separate `decode` function that skips the signature entirely.

The main operational drawback is revocation. A signed token stays valid until `exp`, so a fired employee or a stolen token keeps working after you remove the account. Use short-lived access tokens, often minutes, with a revocable refresh token. A denylist of token IDs also works, but it brings back a per-request lookup.

For ordinary browser logins, a [session cookie](#session-cookie) is usually simpler and can be revoked immediately. If a browser must hold a JWT, avoid `localStorage`, which any injected script can read.

_Usage:_

"We disabled her account an hour ago. Why can she still call the API?"

"Her JWT is valid until it expires. Shorten the access token lifetime and check revocation at refresh."

### OAuth 2.0

OAuth 2.0 is an [authorization](#authorization) framework that lets an application act on a user's resources at another service without receiving the user's password. A scheduling app can request access to your calendar, you approve it on the calendar provider's site, and the app receives a limited token. OAuth calls the parties the resource owner (user), client (app), authorization server (issues tokens) and resource server (the API).

For apps acting on behalf of a user, use the authorization code flow with PKCE (Proof Key for Code Exchange):

| Step | Who                  | What happens                                                     |
| ---- | -------------------- | ---------------------------------------------------------------- |
| 1    | Client               | Creates a secret `code_verifier`, a `state` value and a hash of the verifier |
| 2    | Client               | Redirects the browser with scopes, `state` and the hash          |
| 3    | User                 | Logs in and approves                 |
| 4    | Authorization server | Redirects back with a short-lived, one-time code                 |
| 5    | Client               | Checks `state`, then exchanges the code plus verifier for tokens |
| 6    | Client               | Calls the API with the access token                              |

PKCE ensures that an intercepted code cannot be redeemed without the verifier. The `state` value ties the response to the browser session that started it, which guards against [CSRF](#csrf).

Register redirect URIs and match them exactly, because loose matching lets an attacker receive codes. Request the narrowest scopes. Keep client secrets and refresh tokens on the server. Do not use the implicit or password grants, which current guidance rules out. For service-to-service calls with no user, use the client credentials grant.

OAuth does not by itself tell an app who logged in. An access token is meant for the API and is not proof of identity. For sign-in, use [OpenID Connect](#openid-connect). Access tokens may be opaque or a [JWT](#jwt), and the client should not depend on their format.

_Usage:_

"Can we just use the Google access token to log people in?"

"No, OAuth 2.0 access tokens aren't proof of login. Use OpenID Connect and validate the ID token."

### OpenID Connect

OpenID Connect (OIDC) is an identity layer built on [OAuth 2.0](#oauth-20). OAuth grants an app access to an API, while OIDC also tells the app who the user is. It powers "Sign in with Google" style logins and enterprise single sign-on, where one account works across many applications.

An app requests the `openid` scope during the OAuth code flow. Along with an access token, the identity provider returns an ID token, a [JWT](#jwt) containing claims about the login event.

| Token         | Audience        | Used for                         |
| ------------- | --------------- | -------------------------------- |
| ID token      | Your client app | Establishing who logged in       |
| Access token  | The API         | Calling that API                 |
| Refresh token | Token endpoint  | Getting new tokens without login |

Do not swap their roles. Sending an ID token to an API as a bearer credential, or treating an access token as proof of login, breaks the audience checks that keep tokens from being misused.

Before starting a session, validate the ID token with a maintained library:

| Check       | Requirement                                                     |
| ----------- | --------------------------------------------------------------- |
| Signature   | Verified against the provider's published keys                  |
| `iss`       | Exactly matches the expected issuer                             |
| `aud`       | Contains your client ID                                         |
| `exp`       | Has not passed                                                  |
| `nonce`     | Matches the value you sent, which blocks replayed tokens        |

Identify users by the pair of `iss` and `sub`, and do not use the email address as the key. Emails change, can be reassigned, and may be unverified at some providers. Automatically merging accounts on a matching email is a known route to account takeover: an attacker registers the victim's address at a provider that does not verify it, then signs in as them.

After validation, the app normally creates its own [session cookie](#session-cookie). Local [authorization](#authorization) remains your responsibility, since a valid login says nothing about what the user may do in your system.

_Usage:_

"Two users share a profile after signing in with different providers."

"We're matching on email. With OpenID Connect, key accounts on issuer plus `sub`."

### RBAC

RBAC (role-based access control) is an [authorization](#authorization) model where permissions are attached to roles and users are assigned roles. Instead of granting "can refund orders" to forty people individually, you grant it to `support_lead` and assign that role.

| Role          | `orders:read` | `orders:refund` | `users:manage` |
| ------------- | ------------- | --------------- | -------------- |
| `viewer`      | Yes           | No              | No             |
| `support`     | Yes           | No              | No             |
| `support_lead`| Yes           | Yes             | No             |
| `admin`       | Yes           | Yes             | Yes            |

Code should check permissions and not role names. A check such as `if user.role == "admin"` spreads through the codebase, and adding a new role later means finding every one of those conditions. `require("orders:refund")` keeps the mapping in one place, so a role can change without code changes.

In a product with multiple customers, a role assignment must be scoped to a tenant. A user may be `admin` in one organisation and `viewer` in another. If the role is stored globally on the user, an admin of a small test account can gain admin rights over everyone's data.

RBAC answers "may this kind of user do this kind of thing?" It does not on its own answer "is this their invoice?" You still need an object-level ownership check. The usual symptom of stretching it too far is role explosion: `manager_eu`, `manager_eu_readonly`, `manager_us_contractor`, until nobody can say what each role grants. That is a sign that the rule depends on attributes, which is the territory of [ABAC](#abac). Many systems use roles for broad access and attribute checks for the exceptions.

Apply least privilege: give the smallest role that does the job, review assignments regularly and remove them when people change teams. Log every role change, since it is a security-relevant event.

_Usage:_

"We've got 140 roles and nobody knows what half of them do."

"RBAC is being used to encode region and team. Keep a few roles and move those conditions into attribute rules."

### ABAC

ABAC (attribute-based access control) is an [authorization](#authorization) model where a policy decides access by evaluating attributes. Those attributes describe four things: the subject (user), the resource, the action and the environment. A rule might read: "A manager may approve an expense if it belongs to their department, is under their limit and was not submitted by them."

```text
allow approve on expense if
  user.department == expense.department
  and expense.amount <= user.approval_limit
  and expense.submitter_id != user.id
```

Expressing this with [RBAC](#rbac) alone would need a role per department and limit, which is the role explosion that usually prompts teams to adopt ABAC.

| Aspect            | RBAC                               | ABAC                                   |
| ----------------- | ---------------------------------- | -------------------------------------- |
| Decision based on | Assigned roles                     | Attributes and policy rules            |
| Granularity       | Coarse                             | Fine, down to individual records       |
| Answering "who can access this?" | Read role assignments | Must evaluate policies against users   |
| Implementation effort | Lower                          | Higher                                 |
| Good fit          | Stable job functions               | Ownership, region, time or data class  |

ABAC is only as trustworthy as its attributes. If a user can edit their own `department` through a profile endpoint, they can grant themselves access, so attributes used in decisions must come from a trusted source. Treat a failed attribute lookup or an unmatched rule as a denial.

List endpoints are the usual implementation difficulty. Loading every row and filtering it in application code is slow and risks leaking data through counts or [pagination](#pagination). The policy needs to be translated into a query filter, which some policy engines support directly.

Policy engines such as Open Policy Agent and Cedar keep rules outside application code, so they can be reviewed and tested. Because a small policy edit can widen access silently, add tests for allowed and denied cases and log each decision.

_Usage:_

"Managers should only approve expenses from their own team, and never their own."

"That doesn't fit a role. Write it as an ABAC policy on department and submitter."

### API key

An API key is a long random secret that a program sends to identify itself to an API. It is commonly used for server-to-server integrations and scripts, where no person is present to complete a login. A key provides simple [authentication](#authentication) of the calling project or account, and it usually stays valid until someone revokes it. That long lifetime makes leaks serious.

Key handling by stage:

| Stage    | What to do                                                                     |
| -------- | ------------------------------------------------------------------------------ |
| Create   | Generate at least 128 random bits with a cryptographically secure generator    |
| Display  | Show the full key once, and keep only a short prefix for identification        |
| Store    | Save a hash of the key, never the plaintext                                    |
| Send     | Use a request header, over [TLS](#tls)                     |
| Limit    | Apply scopes, per-key rate limits and optional IP restrictions                 |
| Rotate   | Allow two active keys so clients can switch without downtime                   |
| Revoke   | Disable immediately and record last-used time                                  |

Never put keys in URLs. Query strings end up in access logs, browser history, proxies and `Referer` headers. Never embed one in frontend or mobile code either, since anyone can extract it from the bundle.

Because generated keys have high entropy, a fast hash such as SHA-256 is acceptable for storage, unlike human-chosen passwords, which need slow [password hashing](#password-hashing).

A recognisable prefix, such as `acme_live_`, helps people tell test keys from live keys and lets secret-scanning tools detect leaks. If a key is committed to a repository, revoke it.

An API key identifies an application and not a person. For actions on behalf of individual users, use [OAuth 2.0](#oauth-20). Apply [authorization](#authorization) to every key, so a read-only integration cannot write.

_Usage:_

"Someone pushed our API key to a public repo, but I've already removed the commit."

"That doesn't help. Assume it's compromised, so revoke it, issue a new one and check its usage logs."

### Password hashing

Password hashing is storing the output of a slow, salted, one-way function in place of the password itself. At login, the server applies the same function to the submitted password and compares results. If the database leaks, attackers still have to guess each password individually.

The function is one-way, so the output cannot be reversed. A salt, a random value unique to each password and stored beside the hash, means identical passwords produce different hashes and precomputed tables are useless. The function is deliberately expensive, which limits how many guesses per second an attacker can make.

General-purpose hashes such as MD5, SHA-1 and SHA-256 are designed to be fast, so they are the wrong tool here, even with a salt. Encryption is also wrong, since anyone who obtains the key can recover every password.

OWASP's password storage guidance lists these minimum settings, which you should recheck periodically:

| Algorithm          | Minimum configuration                   | Notes                          |
| ------------------ | --------------------------------------- | ------------------------------ |
| Argon2id           | 19 MiB memory, 2 iterations, p=1 | First choice                   |
| scrypt             | N=2^17, r=8, p=1                        | Use if Argon2id is unavailable |
| bcrypt             | Work factor 10 or more                  | Input limited to 72 bytes      |
| PBKDF2-HMAC-SHA256 | 600,000 iterations                      | For FIPS-compliance needs      |

Use a maintained library and its verify function, which handles salts and constant-time comparison. Its encoded output records the algorithm and parameters, so you can raise the cost later and rehash at the next successful login.

Slow hashing has a server-side cost. A burst of login attempts can saturate the CPU and, in a single-threaded runtime, stall the [event loop](#event-loop) for every user. Rate-limit login, run hashing off the main [thread](#thread), and set a generous maximum password length so very long inputs cannot be used for denial of service.

_Usage:_

"The audit flagged our password column. It's SHA-256 with a salt, isn't that fine?"

"No, it's too fast. Move to Argon2id password hashing and rehash each user at next login."

### CSRF

CSRF (cross-site request forgery) is an attack where a malicious website causes a victim's browser to send a request to another site where they are logged in. Browsers can attach cookies automatically, so the target receives a request carrying a valid [session cookie](#session-cookie) and treats it as genuine. The attacker cannot read the response, but the action still happens.

```html
<form action="https://bank.example/transfer" method="POST">
  <input type="hidden" name="to" value="attacker" />
  <input type="hidden" name="amount" value="500" />
</form>
<script>document.forms[0].submit()</script>
```

Layer these defences for cookie-authenticated, state-changing routes:

| Defence                       | How it works                                                  | Limit                                    |
| ----------------------------- | ------------------------------------------------------------- | ---------------------------------------- |
| Anti-CSRF token               | Server checks an unpredictable per-session token on each write | Must cover every state-changing route   |
| `SameSite=Lax` or `Strict`    | Browser withholds the cookie on cross-site POSTs              | Sibling subdomains count as same-site    |
| `Origin` / `Sec-Fetch-Site`   | Server rejects requests from untrusted origins                | Decide how to treat a missing header     |
| Correct [HTTP methods](#http-method) | GET never changes state                          | `Lax` still sends cookies on top-level GET |

Set `SameSite` explicitly and do not rely on browser defaults, which vary. Use your framework's built-in protection where it exists, since a common failure is a developer disabling it after hitting a "CSRF token missing" error on a form.

CORS (the browser rules for cross-origin reads) is not a CSRF defence. It controls whether a page can read a response, and a simple cross-site form POST is still sent.

APIs that authenticate only with an `Authorization` header are generally not exposed to CSRF, because browsers do not attach that header automatically. Cross-site scripting defeats these protections, since attacker code running on your page can read tokens and make same-origin requests. Protect login forms as well, because login CSRF can sign a victim into the attacker's account.

_Usage:_

"This POST fails with 'CSRF token missing'. Can I exempt the route?"

"No, it changes account settings. Send the token from the form instead of turning off CSRF protection."

### SQL injection

SQL injection is a vulnerability where untrusted input becomes part of the text of a database query, so the database runs it as code. It can expose or alter data and bypass login, and on some configurations it can reach the underlying server. An early, harmless sign is a search or signup form that returns a 500 when someone named O'Brien uses it: the apostrophe has ended the string in the query.

```python
# Vulnerable: input is concatenated into the statement
db.execute(f"SELECT * FROM users WHERE email = '{email}'")

# Safe: the value is sent separately as a bound parameter
db.execute("SELECT * FROM users WHERE email = %s", (email,))
```

With a parameterized query, the driver or database treats the value strictly as data, whatever characters it contains. This is the primary defence. Manual escaping and blocklists of words such as `DROP` are unreliable and should not be used in its place.

Parameters do not cover every part of a query:

| Query part                | Safe approach                                             |
| ------------------------- | --------------------------------------------------------- |
| Values                    | Bound parameters                                          |
| Column or table names     | Map user input to a hardcoded allowlist                   |
| Sort direction            | Allow only `ASC` or `DESC`                                |
| `IN (...)` lists          | Generate one placeholder per value, or bind an array      |
| Raw queries in an [ORM](#orm) | Use the ORM's parameter binding, never interpolation |
| Dynamic SQL in stored procedures | Bind parameters there too                          |

An ORM protects its normal query methods, but its raw-query features can still be injected. Stored data can also be dangerous: in second-order injection, a value saved safely is later concatenated into another query, so treat values read from your own database as untrusted too.

Add defence in depth. Connect with a database account holding the least privileges it needs, so that an injection cannot drop tables. Return generic error messages, since raw database errors help attackers refine a payload. Input validation is useful but does not replace parameters.

_Usage:_

"Search crashes whenever the term contains an apostrophe."

"That's SQL injection waiting to happen. Replace the string formatting with a bound parameter."

### SSRF

SSRF (server-side request forgery) is a vulnerability where an attacker makes your server send a request to a destination they choose. It appears in features that fetch a user-supplied URL: link previews, image imports, PDF generators and [webhook](#webhook) destinations. The request originates inside your network, so it can reach things the attacker cannot reach directly, including admin panels, internal APIs and cloud metadata endpoints.

Many cloud platforms expose instance information at the link-local address `169.254.169.254`, sometimes including temporary credentials. If an "import image from URL" feature fetches that address and returns the body, the attacker may obtain those credentials.

Blocklisting strings such as `localhost` is not sufficient. Attackers can use alternate IP encodings, IPv6 forms, a DNS name resolving to an internal address, or a public URL that redirects inward. Apply layered controls:

| Control                  | What to do                                                                |
| ------------------------ | ------------------------------------------------------------------------- |
| Allowlist                | Permit only known hosts wherever the feature allows it                    |
| Scheme and port          | Allow only `https`, or `http` if required, on expected ports              |
| Resolved IP check        | Resolve the name and reject private, loopback, link-local and mapped addresses |
| Pin the connection       | Connect to the validated IP, which prevents DNS rebinding                 |
| Redirects                | Disable them, or revalidate every hop                                     |
| Network egress           | Run fetchers in an isolated network with outbound rules                   |
| Metadata hardening       | Require session-based metadata access, such as AWS IMDSv2                 |
| Response handling        | Enforce size and time limits, and do not return raw bodies or errors      |

DNS rebinding is why checking a URL once is not enough: the name resolves to a public address during validation and an internal one at fetch time.

Return one generic failure, since differing errors let attackers map internal ports. Internal services should still require [authentication](#authentication) instead of trusting a request because it came from inside.

_Usage:_

"The link preview fetched `169.254.169.254` during the pen test."

"That's SSRF. Validate the resolved IP, pin the connection and block redirects."

## Section 4 — Concurrency

### Concurrency

Concurrency is structuring a program so that several tasks are in progress during the same period. The tasks may take turns on a single CPU core, which distinguishes it from [parallelism](#parallelism), where tasks execute at the same instant. A server needs concurrency because requests overlap: while one waits for a database response, another can use the CPU.

Languages and runtimes offer different models:

| Model                                  | Who switches tasks                  | Examples                          |
| -------------------------------------- | ----------------------------------- | --------------------------------- |
| OS [threads](#thread)              | Operating system, at any moment     | Java platform threads, C++        |
| Runtime-managed threads                | Language runtime                    | Goroutines, Java virtual threads  |
| [Event loop](#event-loop) tasks  | Tasks yield at [await](#async-await) points | Node.js, Python asyncio |
| Multiple processes                     | Operating system, memory isolated   | Pre-fork web servers              |

Concurrency helps most when work spends its time waiting, such as on network calls, disk access or a database. It does little for a task that uses the CPU continuously, because there is no idle time for another task to use.

The difficulty comes from shared state. When tasks overlap, their steps can interleave in orders you did not plan for. That produces the typical concurrency bug: code passes every test with one user, then under load a counter comes up short or two customers book the same seat. Such a [race condition](#race-condition) may appear only rarely, which makes it hard to reproduce.

Protecting shared data with a [mutex](#mutex) solves that but adds waiting, and careless locking can produce a [deadlock](#deadlock). Where possible, avoid sharing mutable state, or let the database enforce the rule atomically.

Limit concurrency as well. Launching one task per item in a list of 100,000 can exhaust connections, memory or a partner's rate limit. A [thread pool](#thread-pool) or semaphore (a counter capping simultaneous tasks) keeps the number bounded.

_Usage:_

"It works perfectly when I test it, but production totals are a few cents off."

"That sounds like a concurrency bug. Two requests are probably updating the same balance at once."

### Parallelism

Parallelism is executing multiple computations at the same instant on separate CPU cores. It is often confused with [concurrency](#concurrency), which is about managing overlapping tasks. A single-core machine can be concurrent by switching between tasks, but it cannot be parallel.

| Situation                         | Concurrent | Parallel | What helps                     |
| --------------------------------- | ---------- | -------- | ------------------------------ |
| 1,000 requests waiting on network | Yes        | Not needed | [Async-await](#async-await) or many [threads](#thread) |
| Resizing images on 8 cores        | Possibly   | Yes      | One task per core              |
| One long sequential calculation   | No         | No       | A faster algorithm             |

Parallelism speeds up CPU-bound work, meaning work limited by computation rather than waiting. Adding cores to an I/O-bound service often changes little, because its time is spent waiting on something else.

Speedup has a limit. Amdahl's law states that the serial portion of a program caps the benefit of adding processors. If 10% of a job must run sequentially, it can never go more than ten times faster, however many cores you add. Coordination, such as waiting on a [mutex](#mutex), adds further cost, and past some point more threads can make a program slower.

Runtimes differ in what they allow. Node.js runs JavaScript on one main thread, so CPU-heavy work needs `worker_threads` or separate processes. CPython's standard build has a global interpreter lock that stops threads running Python code simultaneously, so CPU-bound Python typically uses multiple processes. Optional free-threaded builds have been available since Python 3.13. Go and Java run threads across cores by default.

Containers add a common surprise. A runtime that sizes itself from the host's core count may start far more threads than its CPU limit allows, and the container gets throttled, showing latency spikes while average CPU looks moderate. Check that pool sizes reflect the container's limit.

_Usage:_

"We doubled the cores and the API didn't get any faster."

"It's I/O-bound. Parallelism won't help when every request is waiting on the same database."

### Thread

A thread is an independent path of execution inside a process. Each thread has its own stack and instruction position, while all threads in a process share the same heap memory and open files. Sharing makes communication between them easy and is also the source of most [concurrency](#concurrency) bugs.

The operating system's scheduler decides which threads run and can pause one at nearly any instruction. On a multi-core machine, several run at once, giving [parallelism](#parallelism). Even a single line such as `count += 1` is usually several machine steps, so another thread can run between them and cause a [race condition](#race-condition).

| Unit             | Scheduled by     | Memory                | Relative cost |
| ---------------- | ---------------- | --------------------- | ------------- |
| Process          | Operating system | Isolated              | Highest       |
| OS thread        | Operating system | Shared within process | Medium        |
| Virtual thread or goroutine | Language runtime | Shared, small growable stack | Low |

The traditional server model gives each request its own thread. It is easy to reason about, because code runs top to bottom and [blocking I/O](#blocking-io) only pauses that one thread. Its limit is cost. Each OS thread reserves stack space, and switching between many of them consumes CPU time. A service creating one thread per connection can fail during a traffic spike with an error such as Java's `unable to create native thread`, even though the CPU is barely busy.

Servers therefore reuse a fixed set through a [thread pool](#thread-pool), or use lighter units such as goroutines and Java virtual threads.

Threads in a process share its failures. An uncaught fatal error or memory corruption in one can bring down the entire process. When a service hangs, a thread dump, a snapshot of what each thread is doing, is usually the first diagnostic to take.

_Usage:_

"The service stopped responding, but CPU is at 2%."

"Take a thread dump. If every thread is parked on the same call, we've found what's holding them up."

### Thread pool

A [thread](#thread) pool is a set of long-lived threads that take tasks from a queue and run them. Reusing threads avoids the cost of creating one per task, and the fixed size caps how much work runs at the same time.

| Step | Who         | What happens                                              |
| ---- | ----------- | --------------------------------------------------------- |
| 1    | Caller      | Submits a task                                            |
| 2    | Pool        | Assigns it to an idle thread, or places it in the queue   |
| 3    | Thread      | Runs the task to completion                               |
| 4    | Thread      | Returns to the pool and takes the next task               |
| 5    | Pool        | Rejects the task if the queue is full                     |

The characteristic failure is pool exhaustion. One slow dependency makes every thread wait on it, new requests pile up in the queue, and the entire service appears hung while CPU sits near idle. Endpoints that do not touch the slow dependency also time out, because they cannot get a thread. Set timeouts on every outbound call, and give risky dependencies their own small pool.

Sizing depends on the work. For CPU-bound tasks, start near the number of cores. For tasks that mostly wait on I/O, a larger pool makes sense, but keep it consistent with downstream limits: 200 threads sharing 20 database connections will queue for connections instead.

Bound the queue. An unbounded queue hides overload until memory runs out, and requests wait so long that clients have already given up. Rejecting early with a 503 [status code](#status-code) is usually better.

A task that submits a subtask to the same pool and waits for it can cause a [deadlock](#deadlock) when all threads are doing this. Because threads are reused, thread-local data left by one request, such as a user ID, can leak into the next unless it is cleared.

_Usage:_

"Even the health check is timing out, and the CPU is idle."

"The thread pool's exhausted. Every thread is stuck on that payment call because it has no timeout."

### Event loop

An event loop is a loop that waits for events, such as data arriving on a socket, and runs the code registered for each one. It lets a single [thread](#thread) serve thousands of connections, because a connection that is waiting for data costs no thread. Node.js, Python's asyncio, Nginx and Redis are built around this design.

| Step | Who         | What happens                                                  |
| ---- | ----------- | ------------------------------------------------------------- |
| 1    | Application | Starts an I/O operation and registers what to do on completion |
| 2    | Event loop  | Continues with other ready work                               |
| 3    | OS          | Reports ready sockets through a mechanism such as epoll or kqueue |
| 4    | Event loop  | Queues the matching callback or resumes the paused task       |
| 5    | Event loop  | Runs it until it finishes or yields, then repeats             |

Scheduling is cooperative: nothing interrupts running code. A task keeps the thread until it returns or reaches an `await` in [async-await](#async-await) code. The main failure follows from this: one slow operation freezes every other request in the process. A 2-second `JSON.parse` on a large payload, a synchronous file read, an expensive regular expression or [password hashing](#password-hashing) on the main thread means all users wait 2 seconds.

The symptom is latency rising across all endpoints at once, with one CPU core at 100%. Measure event loop lag, the delay between when a callback was due and when it ran.

To keep the loop responsive, avoid [blocking I/O](#blocking-io) on it. Move CPU-heavy work to a separate thread, a separate process or a [background job](#background-job). Limit request body sizes and split long computations.

"Single-threaded" describes the part that runs your code. Node.js also uses a small internal [thread pool](#thread-pool) for some file, DNS lookup and crypto operations. It has four threads by default, so those operations can queue behind each other.

_Usage:_

"Every endpoint spikes whenever someone downloads the big report."

"The report builder runs on the event loop and blocks it. Move it off the main thread."

### Blocking IO

Blocking I/O is an input or output operation that suspends the calling [thread](#thread) until it completes. A call such as `read()` on a socket does not return until data arrives. While it waits, the thread uses almost no CPU, but it cannot do anything else. Network calls, disk access and database queries are the usual examples.

Non-blocking I/O returns control immediately, and the program learns later that the operation has finished.

| Aspect              | Blocking                         | Non-blocking                               |
| ------------------- | -------------------------------- | ------------------------------------------ |
| Code style          | Sequential and easy to read      | [Async-await](#async-await), callbacks |
| Waiting costs       | One thread per in-flight call    | A small record per in-flight call          |
| Typical server      | [Thread pool](#thread-pool) | [Event loop](#event-loop)           |
| Main risk           | Running out of threads           | One blocking call stalls everyone          |

Blocking is fine in a threaded server, provided the pool is sized for it and every call has a timeout. Without a timeout, a dependency that stops answering holds its thread indefinitely, and enough of those calls will exhaust the pool.

The damaging case is a blocking call inside an event loop. In Node.js, `fs.readFileSync` stops every request in the process while it runs. In Python, calling `time.sleep()` or the synchronous `requests` library inside an `async def` handler freezes the loop in the same way. The code looks correct and passes tests with one user, then throughput collapses under load because requests are handled one at a time.

To fix it, use the async version of the library, or move the call to a separate thread, for example with `asyncio.to_thread` in Python.

Not all I/O has a non-blocking form on every platform. Regular file reads on Linux generally block even when a descriptor is marked non-blocking, so runtimes such as Node.js hand file operations to helper threads.

_Usage:_

"Why does one slow upload stall the whole FastAPI app?"

"That `async def` handler does blocking I/O with `requests`. Use an async client or run it in a thread."

### Async-await

Async-await is language syntax for writing non-blocking code that reads like sequential code. An `async` function can pause at each `await`, handing control back to the scheduler, and resume when the awaited operation completes. JavaScript, Python, C# and Rust all provide it. Usually an [event loop](#event-loop) or similar runtime does the scheduling.

```javascript
async function getOrder(id) {
  const order = await db.findOrder(id); // other tasks run during this wait
  const user = await db.findUser(order.userId);
  return { order, user };
}
```

The compiler or runtime turns such a function into a resumable state machine, so a paused function holds no [thread](#thread). This gives [concurrency](#concurrency) for I/O-bound work. It does not make CPU-heavy code faster or run it in [parallel](#parallelism).

Common mistakes have recognisable symptoms:

| Mistake                         | Symptom                                                 | Fix                                       |
| ------------------------------- | ------------------------------------------------------- | ----------------------------------------- |
| Forgotten `await`               | Code continues early, errors vanish or go unhandled     | Enable a lint rule for floating promises  |
| Awaiting in a loop              | 20 independent calls run one after another              | `Promise.all`, `asyncio.gather`           |
| Unbounded `Promise.all`         | 10,000 calls at once exhaust connections                | Add a concurrency limit                   |
| [Blocking I/O](#blocking-io) in an async function | Every request stalls                 | Use an async client or a separate thread  |
| Fire-and-forget task            | Work is lost on restart, failures go unseen             | Use a [background job](#background-job) |

A common misconception is that single-threaded async code cannot have a [race condition](#race-condition). It can. Other tasks run at every `await`, so a sequence that reads a balance, awaits something, then writes the balance can interleave with another request doing the same. Keep check-and-update logic free of awaits, or make it atomic in the database.

Async also spreads through a codebase: a function that awaits must generally be async itself, and so must its callers. This is why mixing sync and async libraries takes planning.

_Usage:_

"The endpoint makes five API calls and takes five seconds."

"You're using async-await one call at a time. They're independent, so run them with `Promise.all`."

### Race condition

A race condition is a bug where the outcome depends on the timing of overlapping operations. The code is correct when steps run one after another and wrong when they interleave. It explains failures such as an oversold last ticket, a coupon redeemed twice, or a counter that ends up lower than the number of requests. These bugs often pass tests and appear only under production load, because [concurrency](#concurrency) is what exposes them.

One common form is a lost update:

| Step | Request A      | Request B      | Stored balance |
| ---- | -------------- | -------------- | -------------- |
| 1    | Reads 100      |                | 100            |
| 2    |                | Reads 100      | 100            |
| 3    | Writes 100 - 30 |               | 70             |
| 4    |                | Writes 100 - 50 | 50            |

Both requests succeeded, 80 was withdrawn, and the balance shows only 50 removed. The other common form is check-then-act: two requests both check that a username is free, both see that it is, and both create it. When the gap involves a security check, it is called time-of-check to time-of-use (TOCTOU).

Fixes, in rough order of preference:

| Approach                | Example                                                      |
| ----------------------- | ------------------------------------------------------------ |
| Atomic operation        | `UPDATE accounts SET balance = balance - 30 WHERE id = 1 AND balance >= 30` |
| Database constraint     | A unique index on `username`, with the conflict handled      |
| Optimistic check        | Update only if a version column is unchanged, else retry     |
| Lock                    | A [mutex](#mutex) or a row lock around the critical steps |
| Remove sharing          | Process each account's work through one ordered queue        |

An in-process mutex protects only that process. With several server instances, two requests can hold "the" lock at the same time on different machines, so the rule must be enforced in a shared system such as the database.

Single-threaded runtimes are not immune, because tasks interleave at each `await` in [async-await](#async-await) code.

_Usage:_

"We sold 102 tickets for a 100-seat venue."

"That's a race condition in the availability check. Make the decrement atomic and conditional on remaining seats."

### Mutex

A mutex (mutual exclusion lock) is a synchronization primitive that lets only one [thread](#thread) at a time run a protected section of code. A thread locks it before touching shared data and unlocks it afterwards. Any other thread attempting to lock it waits. The protected code is called the critical section, and a mutex is a standard fix for a [race condition](#race-condition) on in-memory state.

```go
mu.Lock()
defer mu.Unlock() // released on every return path
balance -= amount
```

Release the lock on every path, using `defer`, `finally` or a scoped guard. If an exception skips the unlock, every later caller waits forever and that code path hangs.

Locks have a performance cost when contended. Since only one thread can hold the mutex, a busy critical section turns parallel work into a single-file line. The symptom is latency that climbs with load while CPU stays low, because threads spend their time waiting.

Practical rules:

| Rule                                   | Reason                                                     |
| -------------------------------------- | ---------------------------------------------------------- |
| Keep the critical section short        | Reduces waiting                                            |
| Do not do network or disk calls inside | One slow call stalls every thread needing the lock         |
| Lock consistently                      | One unprotected read or write is still a data race         |
| Acquire several locks in a fixed order | Prevents a [deadlock](#deadlock)                       |
| Do not re-lock unless it is reentrant  | Go's `sync.Mutex` deadlocks if the holder locks it again   |

A read-write lock lets many readers in together but gives a writer exclusive access. A reentrant lock, such as Java's `synchronized`, can be taken again by the thread already holding it.

A mutex works only inside one process. It does nothing for two server instances, so rules across instances belong in the database or another shared coordinator. In single-threaded [async-await](#async-await) code, an async mutex can protect a sequence that spans an `await`.

_Usage:_

"Latency goes up with traffic, but the CPU's only at 20%."

"Check the mutex around the cache. It's held during a network call, so every request queues behind it."

### Deadlock

A deadlock is a state in which two or more tasks each wait for something another holds, so none of them can ever proceed. The typical symptom is a service that stops responding with low CPU and no error in the logs, because nothing has failed.

The simplest case involves two locks:

| Step | Task A           | Task B           |
| ---- | ---------------- | ---------------- |
| 1    | Locks `accounts` |                  |
| 2    |                  | Locks `orders`   |
| 3    | Waits for `orders` |                |
| 4    |                  | Waits for `accounts` |

A deadlock requires four conditions at once, known as the Coffman conditions, and removing any one prevents it:

| Condition        | Meaning                                  | How to break it                          |
| ---------------- | ---------------------------------------- | ---------------------------------------- |
| Mutual exclusion | A resource has one holder at a time      | Use lock-free or immutable data          |
| Hold and wait    | A task holds one resource, requests more | Acquire everything at once               |
| No preemption    | A resource cannot be taken away          | Use lock timeouts and back off           |
| Circular wait    | Tasks wait in a cycle                    | Always acquire in one global order       |

Consistent ordering is the usual fix. If every code path takes a [mutex](#mutex) on the lower account ID first, a transfer from A to B and one from B to A cannot form a cycle.

Deadlocks also occur without explicit locks. A [thread pool](#thread-pool) deadlocks when every [thread](#thread) in it waits for subtasks queued in that same pool. A request holding one database connection while waiting for a second can deadlock a small connection pool under load. Calling `.Result` on a task in some C# contexts blocks the thread that the task needs in order to finish.

To diagnose one, take a thread dump and look for tasks waiting on each other. Relational databases detect lock cycles and abort one transaction, so applications should retry that error. Timeouts turn an indefinite hang into a visible failure.

_Usage:_

"Transfers hang now and then, always between the same two accounts."

"Opposite transfers lock in opposite order, so it's a deadlock. Always lock the lower account ID first."

## Section 5 — Background Work

### Background job

A background job is a unit of work that runs outside the [request lifecycle](#request-lifecycle). The handler records that the work must happen, responds to the user and leaves execution to another process. Sending email, generating reports, resizing images and syncing with third parties are typical cases.

The symptom that calls for one is a request doing too much. Signup takes eight seconds because it calls an email provider, and when that provider has an outage, signups start failing even though the account could have been created. Moving the email to a job keeps the endpoint fast and isolates the failure.

Jobs are usually placed on a [job queue](#job-queue) and executed by a [worker](#worker). If the client needs the outcome, the API can return `202 Accepted` with a job ID to check, or notify it through a [webhook](#webhook) or [Server-Sent Events](#server-sent-events).

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

### Job queue

A job queue is a store of pending [background jobs](#background-job). Application code adds jobs, and [workers](#worker) take them and run them. The queue separates when work is requested from when it is done, so traffic bursts build up as a backlog instead of overloading the system. Sidekiq, BullMQ and Celery are examples.

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

### Worker

A worker is a long-running process that repeatedly takes a [background job](#background-job) from a [job queue](#job-queue), runs it and reports the result. Workers usually share the application's codebase but run as separate processes from the web servers, so heavy jobs do not take capacity from user requests and each side can scale independently.

A worker has its own [concurrency](#concurrency) setting, the number of jobs it runs at once. The right value depends on the work:

| Job type            | Suggested setup                                     | Reason                             |
| ------------------- | --------------------------------------------------- | ---------------------------------- |
| CPU-heavy           | About one job per core                              | More only adds switching           |
| I/O-heavy           | Higher concurrency                                  | Jobs mostly wait                   |
| Memory-heavy        | Low concurrency, dedicated workers                  | Prevents out-of-memory kills       |
| Rate-limited API    | A limit shared across workers                       | Per-process limits add up          |

Adding workers is not always safe. Each one opens connections, so doubling the count can exhaust the database's connection limit and turn a slow backlog into errors for the web application too. Check downstream capacity before scaling, and scale on queue age or depth instead of CPU when jobs mostly wait.

Prefetching causes another surprise. Some systems let a worker reserve several jobs in advance. A worker holding a batch while stuck on one long job leaves the others waiting even though other workers are idle. Lower the prefetch setting for long-running jobs.

Deploys need care because old and new code run at the same time. A new web server can enqueue a job in a format an old worker cannot read, so deploy changes to job payloads in compatible steps. Workers also need [graceful shutdown](#graceful-shutdown): on a stop signal, stop taking new jobs and finish or release current ones.

_Usage:_

"We added ten workers to clear the backlog and now the API is throwing connection errors."

"Each worker holds its own connections, so we've hit the database limit. Scale back and lower their concurrency."

### Cron job

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

Write the job so that running it twice is harmless, for example by recording which billing period has been processed. Keep the cron entry small: for large tasks, have it enqueue a [background job](#background-job) on a [job queue](#job-queue), which provides retries and visibility.

_Usage:_

"Customers got three copies of the monthly invoice."

"The cron job runs on every instance, and we scaled to three. Move it to a single scheduler and make it idempotent."

### Graceful shutdown

Graceful shutdown is stopping a process in an orderly way: it refuses new work, finishes what is in progress, closes its resources and exits. Without it, each deploy kills in-flight requests, which appears as a burst of 502 errors at every release.

Platforms normally signal a process with `SIGTERM`, wait for a grace period, then force it to stop with `SIGKILL`, which cannot be caught. Kubernetes, for example, waits 30 seconds by default. A server should then:

| Step | Who     | What happens                                                                 |
| ---- | ------- | ---------------------------------------------------------------------------- |
| 1    | Process | Receives `SIGTERM`                                                           |
| 2    | Process | Fails its readiness check so traffic stops arriving       |
| 3    | Process | Waits briefly, because routing updates are not instant                       |
| 4    | Process | Stops accepting connections and closes idle [keep-alive](#keep-alive) ones |
| 5    | Process | Lets in-flight requests finish, up to a deadline                             |
| 6    | Process | Closes database pools and flushes logs and metrics                           |
| 7    | Process | Exits before the grace period ends                                           |

Step 3 is often missed. If the server closes its listener immediately, the load balancer (the component spreading traffic across instances) can still route requests to it for a few seconds, and those requests fail.

A [WebSocket](#websocket) or [Server-Sent Events](#server-sent-events) stream never finishes alone, so the server must close it and rely on clients reconnecting with random delay.

A [worker](#worker) stops reserving jobs, then finishes the current job or returns it to the [job queue](#job-queue). If a job runs longer than the grace period, it will be killed, so it must be safe to retry.

A common container bug blocks this. If the image starts the app through a shell, as in `CMD node server.js`, the shell may receive `SIGTERM` and not forward it. The app is then killed at the deadline. Use the exec form, `CMD ["node", "server.js"]`.

_Usage:_

"We get a spike of 502s on every deploy."

"The app has no graceful shutdown. Handle `SIGTERM`, fail readiness, then drain before exiting."

## Section 6 — Code Structure

### Layered architecture

Layered architecture is a way of organising an application into horizontal layers, each with one kind of responsibility. Code in a layer calls the layer beneath it and knows nothing about the layers above. It is the default structure of many web frameworks.

| Layer        | Responsibility                               | Typical contents             |
| ------------ | -------------------------------------------- | ---------------------------- |
| Presentation | Handle [HTTP](#http), parse input, format output | Controllers, [DTOs](#dto) |
| Application  | Coordinate a use case and its transaction    | Services                     |
| Domain       | Hold business rules                          | Entities, calculations       |
| Data access  | Read and write storage                       | [Repositories](#repository-pattern), [ORM](#orm) models |

The purpose is to separate reasons for change. A new response format should affect the presentation layer, and a query optimization should affect data access. The structure also gives a team a shared answer to "where does this code go?"

Layering breaks down in recognisable ways. SQL appears in a controller, so the same rule gets reimplemented in three endpoints with slight differences. Business logic lands in controllers, so a [background job](#background-job) that needs the same rule has to copy it, because it has no HTTP request to pass through. Keeping rules below the presentation layer lets every entry point reuse them.

The opposite failure is ceremony. A service method that only calls a repository method, which only calls the ORM, adds files without adding behaviour. Adding one field then means editing every layer. For a small CRUD service (one that mainly creates, reads, updates and deletes records), fewer layers are reasonable.

A further limitation is the direction of dependencies. In the traditional form, business logic depends on the data layer, so database details can leak upward and the rules are hard to test alone. [Hexagonal architecture](#hexagonal-architecture) inverts that dependency using interfaces and [dependency injection](#dependency-injection).

_Usage:_

"Where should the discount calculation go? It's in the controller right now."

"Move it down. In our layered architecture, the [worker](#worker) and the API should both call the same domain code."

### Hexagonal architecture

Hexagonal architecture, also called ports and adapters, is a structure that keeps business logic independent of technical details. Alistair Cockburn proposed it. The application sits at the centre and communicates with the outside world only through interfaces called ports. Adapters implement those ports for specific technologies.

| Part            | What it is                                  | Example                            |
| --------------- | ------------------------------------------- | ---------------------------------- |
| Application     | Use cases and business rules                | `PlaceOrder`                       |
| Inbound port    | How the outside calls the application       | `PlaceOrder.execute(command)`      |
| Inbound adapter | Translates an external trigger into a call  | [HTTP](#http) controller, queue consumer |
| Outbound port   | What the application needs from outside     | `PaymentGateway` interface         |
| Outbound adapter | Implements that need with a technology     | Stripe client, Postgres [repository](#repository-pattern) |

The defining rule is that dependencies point inward. The application defines `PaymentGateway`, and the payment adapter implements it. This differs from traditional [layered architecture](#layered-architecture), where business code usually depends directly on the data layer. Adapters are connected at startup, normally through [dependency injection](#dependency-injection).

The structure addresses a familiar problem: business logic that cannot be tested without a running database and a sandbox payment account, or a vendor's SDK types spread across hundreds of files so that changing vendors becomes a rewrite. With ports, a [unit test](#unit-test) can run a use case with a [test double](#test-double) for each outbound port. The same use case can be triggered by HTTP, a [cron job](#cron-job) or a [worker](#worker) by adding an adapter.

The cost is indirection. Each external interaction needs an interface, an adapter and mapping code, which is unnecessary for a thin CRUD application with few rules. A port that mirrors a vendor's API keeps the coupling it was supposed to remove, so define ports in terms of what the application needs. Adapters still need an [integration test](#integration-test), since test doubles cannot show that the real technology behaves as assumed.

_Usage:_

"Replacing the payment provider means touching 200 files."

"Because the SDK leaked everywhere. With hexagonal architecture, we'd only write a new adapter for the payment port."

### Dependency injection

Dependency injection (DI) is passing a component the objects it needs from the outside, instead of having it create or look them up itself.

```typescript
// Hard-wired: always uses the real client
class SignupService {
  private mailer = new SmtpMailer(process.env.SMTP_URL);
}

// Injected: the caller decides which mailer to use
class SignupService {
  constructor(private mailer: Mailer) {}
}
```

The first version shows the usual symptom. To test signup you need a real mail server, or you must patch a module at runtime, and the test sends real email if configuration goes wrong. With injection, a [unit test](#unit-test) passes a [test double](#test-double), and production passes the real implementation. The class depends on an interface and on nothing else.

| Form        | How it is supplied         | When to use                                 |
| ----------- | -------------------------- | ------------------------------------------- |
| Constructor | Argument when created      | Default choice, since nothing can be missing |
| Method      | Argument on each call      | The value varies per call                   |
| Property    | Assigned after creation    | Optional dependencies, framework constraints |

Dependency injection does not require a framework. Wiring objects by hand in one startup location, called the composition root, is often enough. Containers such as those in Spring, NestJS and ASP.NET Core automate this wiring, which helps in large applications but can make it harder to trace where an object came from. Errors may also appear only at startup or at first use.

Containers introduce lifetime bugs. Objects can be created once per application or once per request. If a singleton captures a request-scoped object, it keeps the first request's data, and one user's context can appear in another user's response. Check that long-lived objects do not hold short-lived ones.

A constructor with ten parameters usually indicates a class doing too much, and DI has made that visible. DI is the mechanism that connects adapters to ports in [hexagonal architecture](#hexagonal-architecture). Simple pure functions and value objects do not need it.

_Usage:_

"I can't test this service without it sending a real email."

"It builds its own mail client. Use dependency injection so the test can pass in a fake."

### Repository pattern

The repository pattern places data access behind an interface that resembles a collection of domain objects. Business code calls methods such as `orders.findById(id)` or `orders.save(order)` and does not need to know how the data is stored. The pattern was described in Martin Fowler's catalogue of enterprise patterns and is a building block of [domain-driven design](#domain-driven-design).

```typescript
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findOverdue(asOf: Date): Promise<Order[]>;
  save(order: Order): Promise<void>;
}
```

It addresses query logic scattered across a codebase. When the definition of an "active customer" appears in fifteen handwritten queries, a change to the rule leaves some of them behind, and reports begin to disagree. A named method such as `findActive()` puts the definition in one place. The interface also lets a [unit test](#unit-test) use an in-memory [test double](#test-double), and it serves as an outbound port in [hexagonal architecture](#hexagonal-architecture).

The pattern is often misapplied:

| Misuse                          | Result                                                  | Better approach                       |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------- |
| Generic `Repository<T>` for all | A thin wrapper repeating the [ORM](#orm)            | Methods named for real queries        |
| Returning query builders        | Storage details leak to callers                         | Return domain objects                 |
| Loops calling `findById`        | One query per item, slow lists                          | Add a batch method                    |
| One method per screen           | The interface grows without limit                       | Use separate read queries             |
| Each repository commits alone   | A multi-step operation can half-succeed                 | Manage the transaction at the use case |

Transactions need attention. If `orders.save()` and `inventory.save()` commit separately, a failure between them leaves inconsistent data. Have the application layer open one transaction, sometimes through a unit-of-work object, that both repositories join.

An in-memory fake will not reproduce real constraints, collation or locking, so test the actual repository with an [integration test](#integration-test). For small applications where an ORM already provides a clear interface, adding repositories may only add a layer.

_Usage:_

"Finance and support are showing different counts of active customers."

"Each has its own query. Put the rule in one repository method and use it in both."

### DTO

A DTO (data transfer object) is a simple object that carries data across a boundary, such as between an API and application code. It holds fields and has no business behaviour. Its purpose is to make the shape of data at the boundary explicit and independent from internal models.

Without a DTO, handlers often serialize the database model directly. This causes two well-known security failures:

| Failure            | What happens                                                            | DTO fix                       |
| ------------------ | ----------------------------------------------------------------------- | ----------------------------- |
| Data exposure      | A new `password_hash` or `internal_notes` column appears in API responses | Response DTO lists allowed fields |
| Mass assignment    | A client adds `"is_admin": true` to a profile update and it gets saved  | Request DTO omits that field  |

Both happen quietly. Nobody edits the endpoint: someone adds a column, and the API starts returning it. An explicit allowlist of fields makes exposure a deliberate decision and supports field-level [authorization](#authorization).

```typescript
type CreateUserRequest = { email: string; name: string };
type UserResponse = { id: string; email: string; name: string };
```

Separate DTOs for requests and responses are normally clearer, because their fields differ. A client should not send an `id` or `createdAt`, while a response contains them.

DTOs also protect the public contract. If a response is tied to an [ORM](#orm) entity, renaming a column becomes a [breaking change](#breaking-change) for API clients. With a DTO, you change the mapping and leave the API alone.

Validate request DTOs at the edge of the system. Check types, required fields and sizes before the data reaches business code, using a schema library where possible. Remember that a type annotation alone usually has no runtime effect, as with TypeScript types, so incoming JSON still needs validation.

The cost is mapping code and some repetition. For an internal tool with few fields, that can outweigh the benefit, but a public API usually justifies it.

_Usage:_

"Why is `password_hash` in the user API response?"

"The controller returns the ORM entity directly. Add a response DTO that lists only the public fields."

### ORM

An ORM (object-relational mapper) is a library that maps database tables to classes and rows to objects, and generates the SQL to read and write them. Hibernate, Entity Framework, SQLAlchemy and Active Record are common examples. Instead of writing an `INSERT`, you create an object and save it.

An ORM removes repetitive query and mapping code, and its standard query methods use bound parameters, which protects them from [SQL injection](#sql-injection). Many also track changes to loaded objects and write them in one transaction.

The trade-off is that database work becomes less visible. A line that looks like property access can run a query, and this leads to the failures most ORM users eventually meet:

| Problem           | Symptom                                               | What to do                                |
| ----------------- | ----------------------------------------------------- | ----------------------------------------- |
| N+1 queries       | A page of 50 orders runs 51 queries                   | Eager-load the relation or batch it       |
| Over-fetching     | A list loads large text columns it never shows        | Select only needed columns                |
| Lazy load after close | Error when serializing outside the session        | Load data before leaving the transaction  |
| Bulk updates      | Hooks and validations are skipped                     | Know which APIs bypass model logic        |

N+1 usually passes in development with three rows, then becomes a slow endpoint in production. Turn on SQL logging locally, and consider a test that asserts the number of queries for important endpoints.

Use raw SQL when the ORM gets in the way, such as for reports, window functions or bulk changes. Mixing the two is normal. Raw queries must still use bound parameters.

Avoid passing ORM entities through the entire application. Exposing them from an API couples your contract to your table structure, which is what a [DTO](#dto) prevents. A [repository](#repository-pattern) can keep ORM details out of business logic.

_Usage:_

"This endpoint is fine locally but takes four seconds in production."

"Turn on query logging. I'd bet the ORM is lazy-loading the customer for each order."

### Domain-driven design

Domain-driven design (DDD) is an approach to software that shapes code around a shared model of the business problem. Eric Evans introduced it in his 2003 book of the same name. Developers work with domain experts, such as accountants or logistics planners, so that the concepts in the code match how the business operates.

It targets a recognisable problem. The business says "a policy lapses", the code says `status = 3`, and every requirement has to be translated in someone's head. DDD answers this with a ubiquitous language: an agreed vocabulary used in conversation, documents, class names and method names, such as `policy.lapse()`.

DDD has two levels. The strategic level divides a large domain into parts, each with a [bounded context](#bounded-context) in which terms have one precise meaning. The tactical level gives patterns for modelling inside a context:

| Building block | Meaning                                              | Example                  |
| -------------- | ---------------------------------------------------- | ------------------------ |
| Entity         | Has an identity that persists as data changes        | `Order #42`              |
| Value object   | Defined by its values, usually immutable             | `Money(10, "EUR")`       |
| Aggregate      | A cluster changed as a unit through one root         | `Order` and its lines    |
| Domain event   | A record of something that happened                  | `OrderShipped`           |
| [Repository](#repository-pattern) | Loads and saves aggregates | `OrderRepository` |

Aggregates protect business rules. If an order total must equal the sum of its lines, all changes go through the `Order` root, so no code can update one line and leave the total wrong.

DDD has a real cost and suits complex, long-lived business domains. A simple CRUD application gains little from aggregates and domain services. A common failure is adopting the tactical patterns without talking to domain experts, which adds classes without improving the model. It pairs well with [hexagonal architecture](#hexagonal-architecture), which keeps the model free of infrastructure.

_Usage:_

"The code calls it `status 3`, but operations calls it 'on hold pending review'."

"Domain-driven design would have us use their term in the code, so we stop translating."

### Bounded context

A bounded context is an explicit boundary inside which a model and its vocabulary have one consistent meaning. The term comes from [domain-driven design](#domain-driven-design). It accepts that a large business does not have one universal model, because the same word means different things to different departments.

| Context  | What "customer" means               | Fields it cares about           |
| -------- | ----------------------------------- | ------------------------------- |
| Sales    | A lead or account being pursued     | Stage, owner, expected value    |
| Billing  | The party that pays invoices        | Tax ID, payment method, balance |
| Shipping | A recipient at an address           | Address, delivery instructions  |
| Support  | A person with open tickets          | Plan, ticket history            |

Without boundaries, these meanings merge into a single model. The symptom is a `User` or `Customer` class with a hundred fields and many nullable columns, which every team edits and nobody fully understands. A billing change breaks shipping, and developers argue over what "active" means because each team is correct about its own area.

Inside each bounded context, the model contains only what that context needs. Billing has its own `Customer`, shipping has its own `Recipient`, and they relate through a shared identifier. Some duplication is intended, since it allows each model to change independently.

Contexts communicate through explicit contracts, such as an API, a [webhook](#webhook) or published events. When one integrates with a messy or external model, it can use an anti-corruption layer, code that translates the foreign model at the edge so its concepts do not leak inward. A [DTO](#dto) often carries the data across this boundary.

A bounded context is a design boundary and does not have to be a separate service. It can be a module inside one deployable application, which is often a sensible starting point. Enforce it by not reaching into another context's tables or internal classes.

_Usage:_

"Should we add `shipping_notes` to the main `Customer` model?"

"No, that belongs to the shipping bounded context. Keep it on `Recipient` and link by customer ID."

## Section 7 — Testing

### Unit test

A unit test is an automated test that checks a small piece of behaviour in isolation. It runs in memory, with no network or real database, so a suite of thousands can finish in seconds and give feedback while you code. Teams disagree on the size of a "unit": it may be one function, one class or a small group of collaborating objects.

```python
def test_discount_is_capped_at_fifty_percent():
    order = Order(subtotal=Money(100))        # arrange
    order.apply_discount(percent=80)          # act
    assert order.total == Money(50)           # assert
```

Unit tests suit logic with many cases: pricing, validation, parsing, date handling and state transitions. Boundary values such as zero, empty input, the maximum and the day of a leap year are cheap to cover here and expensive to cover any other way.

Good unit tests share a few properties:

| Property      | What it means                                      |
| ------------- | -------------------------------------------------- |
| Fast          | Milliseconds each, so people run them constantly   |
| Deterministic | Same result every run, with clock and randomness controlled |
| Isolated      | No dependence on other tests or their order        |
| Behavioural   | Asserts outcomes, not internal steps               |
| Readable      | The name states the rule being protected           |

The common failure is coupling tests to implementation. A test asserting that a private helper was called twice breaks when you refactor, even though the behaviour is unchanged. After enough false alarms, people stop trusting the suite and refactoring slows down. Overuse of a mocking [test double](#test-double) is the usual cause. Prefer checking return values and resulting state, and use doubles mainly for slow or external dependencies.

Unit tests cannot show that components work together. A query can be mocked successfully and still contain invalid SQL, which is what an [integration test](#integration-test) is for. The [test pyramid](#test-pyramid) describes how to balance these levels. Code that is difficult to unit test often has hard-wired dependencies, which [dependency injection](#dependency-injection) can loosen.

_Usage:_

"I renamed one private method and 40 unit tests failed."

"They're asserting on internals. Rewrite them to check the output, so a refactor doesn't break them."

### Integration test

An integration test checks that your code works correctly with a real collaborator, such as a database, message broker or another module. A [unit test](#unit-test) replaces those dependencies, so it cannot catch a mistaken assumption about how they behave. Integration tests catch failures that appear at the join.

The familiar symptom is a green test suite followed by a production failure. The mocked [repository](#repository-pattern) returned a row, but the real query had a typo. The code inserted a duplicate that the real unique constraint rejected. A timestamp lost precision, a migration was missing, or JSON serialized differently from what the test assumed.

Good targets include:

| Target                                     | What it can catch                                  |
| ------------------------------------------ | -------------------------------------------------- |
| Repository and [ORM](#orm) queries | Invalid SQL, constraints, N+1, collation |
| [HTTP](#http) route through [middleware](#middleware) | Routing, [authentication](#authentication), [status codes](#status-code) |
| [Job queue](#job-queue) flows        | Serialization, retries, acknowledgement            |
| Transaction behaviour                      | Rollbacks and isolation between operations         |
| Client for an external API                 | Parsing, timeouts and error mapping, using a local stub server |

Use the same database engine as production. Running tests on SQLite while production uses PostgreSQL hides differences in types, locking and SQL dialect. Tools such as Testcontainers start a real database in a container, disposable per test run.

Isolation is the main practical challenge. Tests sharing a database can interfere and produce a [flaky test](#flaky-test) that only fails when run in a certain order or in parallel. Common solutions include wrapping each test in a transaction that rolls back, giving each test process its own schema, or generating unique data per test.

Integration tests are slower than unit tests, so keep them focused on the boundary and leave detailed business-rule permutations to faster tests. For a dependency owned by another team, a [contract test](#contract-test) is often more practical than running their full system.

_Usage:_

"All the tests pass, but the deploy fails on a unique constraint."

"The repository is mocked everywhere. Add an integration test against a real Postgres container."

### Contract test

A contract test verifies that two services agree on the format of their communication, without deploying them together. The contract records the requests a consumer sends and the responses it depends on. Each side then tests against that record independently.

It addresses a specific gap. Team A's tests mock Team B's API and pass. Team B's tests pass too. Then B renames a field, both pipelines stay green, and the failure appears only after deployment. Each side was testing against its own assumption. A contract test turns that mismatch into a failing build before the [breaking change](#breaking-change) ships.

In consumer-driven contract testing, the flow is as follows:

| Step | Who      | What happens                                                           |
| ---- | -------- | ---------------------------------------------------------------------- |
| 1    | Consumer | Writes a test declaring the request it sends and the fields it needs   |
| 2    | Tool     | Runs that test against a mock and generates a contract file            |
| 3    | Consumer | Publishes the contract to a shared location                            |
| 4    | Provider | Replays the contract against its real code in CI, the automated build  |
| 5    | Both     | Check compatibility before deploying                                   |

Pact is a widely used tool for this approach. When every consumer publishes a contract, a provider can see which fields are unused.

An alternative is schema-based testing, where both sides are checked against a shared [OpenAPI](#openapi) document or [Protocol Buffers](#protocol-buffers) definition. This is simpler to adopt and suits public APIs with unknown consumers, but it shows only that a schema is compatible, and it says nothing about which fields clients rely on.

Contract tests check structure and agreed interactions. They do not verify business logic, so a response with the correct shape and a wrong price still passes. Keep those checks in the provider's [unit tests](#unit-test) and [integration tests](#integration-test).

_Usage:_

"Their deploy broke our checkout again, and both CI pipelines were green."

"We're mocking their API from memory. Add a contract test so their build fails if they drop a field we use."

### Test double

A test double is an object that replaces a real dependency during a test, in the way a stunt double replaces an actor. Gerard Meszaros coined the term in his book _xUnit Test Patterns_. Doubles let a [unit test](#unit-test) avoid sending real email, charging a real card or waiting on a network, and they make hard-to-trigger cases, such as a timeout, easy to simulate.

"Mock" is commonly used for all of them, but there are five distinct kinds:

| Kind  | What it does                                  | Example                                  |
| ----- | --------------------------------------------- | ---------------------------------------- |
| Dummy | Fills a parameter and is never used           | A placeholder logger                     |
| Stub  | Returns canned answers                        | `getRate()` always returns `1.2`         |
| Spy   | Records how it was called                     | Captures emails "sent" for later checks  |
| Mock  | Is preprogrammed with expected calls and verifies them | Fails unless `charge(100)` is called once |
| Fake  | Has a working but simplified implementation   | An in-memory [repository](#repository-pattern) |

Choosing the right kind affects how brittle a suite becomes. Stubs and fakes support checking outcomes. Mocks verify interactions, which fits cases where the call itself is the behaviour, such as "the payment was charged exactly once". Using mocks for every collaborator ties tests to implementation, so a harmless refactor turns the suite red.

The larger risk is drift. A double encodes your belief about how the dependency behaves, and it keeps passing after the real one changes. The stub returns `{ "amount": 100 }` while the real API now returns a string, so tests stay green and production fails. An [integration test](#integration-test) or [contract test](#contract-test) keeps those beliefs honest.

Double what you own, by wrapping a third-party SDK in a small interface of your own and replacing that interface. Inject doubles through [dependency injection](#dependency-injection) instead of patching globals, which can leak between tests and cause a [flaky test](#flaky-test).

_Usage:_

"The payment test passes, but real charges are failing."

"Our test double still returns the old response format. Add a contract test against the provider."

### Test pyramid

The test pyramid is a model for deciding how many tests to write at each level. Mike Cohn popularised it. Its shape reflects cost: the base holds many fast, cheap tests, and each level above holds fewer, slower and more realistic ones.

| Level                                     | Quantity | Speed    | Confidence given             | Failure points to     |
| ----------------------------------------- | -------- | -------- | ---------------------------- | --------------------- |
| End-to-end                                | Few      | Slow     | The full system works        | Anywhere              |
| [Integration](#integration-test)    | Some     | Medium   | Components work together     | A specific boundary   |
| [Unit](#unit-test)                  | Many     | Fast     | Logic is correct             | A small area of code  |

An end-to-end test drives the deployed system as a user or client would. It catches wiring problems nothing else can, but it is slow, can fail for unrelated reasons, and a failure reveals little about the cause.

The usual rule is to test each behaviour at the lowest level that can catch its bug. Cover all discount permutations in unit tests. Use one integration test to show that the discount persists, and one end-to-end test to show that checkout works.

The recognisable anti-pattern is the inverted pyramid, or "ice cream cone". Most coverage sits in browser or end-to-end tests. CI, the automated build, takes an hour, a [flaky test](#flaky-test) fails most runs, and developers rerun the pipeline until it passes or stop reading the results. When a test fails at the top and no lower test failed with it, consider adding the missing lower-level test.

The pyramid is a guideline and not a fixed ratio. A thin API over a database has little isolated logic, so integration tests may provide most of the value. That view is reflected in alternatives such as the "testing trophy". A [contract test](#contract-test) can replace many slow cross-service end-to-end tests.

_Usage:_

"CI takes 70 minutes, and most of that is Selenium."

"Our test pyramid is upside down. Keep a few critical journeys and push the rest down to API and unit tests."

### Flaky test

A flaky test is a test that sometimes passes and sometimes fails on identical code. It often appears as a test that passes alone and fails in CI, the automated build, or a pipeline that goes green when someone presses "re-run". The damage spreads beyond the single test: once people expect random failures, they start ignoring real ones.

Most flakiness has a small set of causes:

| Cause            | Example                                                | Fix                                         |
| ---------------- | ------------------------------------------------------ | ------------------------------------------- |
| Timing           | `sleep(1)` before checking a [background job](#background-job) | Poll for the condition with a timeout |
| Shared state     | Tests reuse one database row or global variable        | Isolate data per test                       |
| Test order       | One test depends on another's leftovers                | Run in random order to expose it            |
| Clock            | Fails at midnight, month-end or in another timezone    | Inject a fixed clock                        |
| Unordered results | Asserting on a query with no `ORDER BY`               | Sort, or compare as a set                   |
| Network          | Calls a real third-party sandbox                       | Use a [test double](#test-double) or stub server |
| Code bug         | A [race condition](#race-condition) in the code  | Fix the production bug                      |

The last row is why flakiness should not be dismissed. A test failing once in fifty runs may be correctly detecting a race that also hits customers.

To reproduce one, run it in a loop, run the suite with the same order seed, and limit CPU to mimic CI.

Automatic retries keep a pipeline moving, but they also hide the problem. If you use them, record every test that passed only on retry. Quarantine a known flaky test by moving it out of the blocking suite, assign an owner and set a deadline.

[Integration tests](#integration-test) and end-to-end tests are more prone to this, which is one reason the [test pyramid](#test-pyramid) keeps them fewer.

_Usage:_

"Just re-run it, that test always fails on Fridays."

"That's a flaky test with a date bug. Inject the clock and fix it instead of re-running."

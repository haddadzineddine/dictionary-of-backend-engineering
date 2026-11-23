---
description: A remote procedure call framework using HTTP/2 and Protocol Buffers, common for typed communication between internal services.
---

gRPC is a framework for calling a function on another service as if it were a local method. You define services and messages in a [Protocol Buffers](./Protocol%20Buffers.md) file, then generate client and server code for your languages. Calls run over [HTTP](./HTTP.md)/2 and their payloads are compact binary.

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

It is most common between internal services, where shared schemas and generated clients prevent mismatched payloads. Browsers cannot speak native gRPC because their APIs do not expose the required HTTP/2 control, so browser clients need gRPC-Web or a translating proxy. For public APIs, [REST](./REST.md) remains easier for outside developers to consume.

Deadlines are a standard feature and should be set on every call. A deadline says how long the caller will wait, and it propagates to downstream gRPC calls when the context is passed along. A service with no deadline can hang indefinitely on a stuck dependency, tying up its own capacity while it waits.

A frequent production surprise concerns load balancing. Because HTTP/2 multiplexes many calls over one long-lived connection, a balancer working at the connection level sends all of a client's calls to one instance. New instances then sit idle while one is overloaded. Use a balancer that understands individual requests, or client-side balancing.

gRPC reports errors with its own codes, such as `UNAVAILABLE` and `DEADLINE_EXCEEDED`, which are separate from HTTP [status codes](./Status%20code.md).

_Usage:_

"We scaled to six pods but one is still taking all the traffic."

"gRPC keeps one HTTP/2 connection open, so the L4 balancer pins it to one pod. We need request-level balancing."

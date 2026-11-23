---
description: The request-and-response protocol that clients and servers use to exchange data on the web and between backend services.
---

HTTP (Hypertext Transfer Protocol) is the protocol a client and server use to exchange messages. The client sends a request containing an [HTTP method](./HTTP%20method.md), a path, headers and an optional body. The server returns a response containing a [status code](./Status%20code.md), headers and an optional body.

HTTP is stateless, which means the protocol keeps no memory of earlier requests. Each request must carry whatever the server needs to process it. Logged-in behaviour is built on top, usually by sending a [session cookie](./Session%20cookie.md) or a [JWT](./JWT.md) every time. This explains a common surprise: a call that works in the browser returns 401 when copied into a script, because the script did not send the credentials the browser was attaching.

The protocol has three versions in common use:

| Version  | Transport     | How requests share a connection                                 |
| -------- | ------------- | --------------------------------------------------------------- |
| HTTP/1.1 | TCP           | One response at a time, in order, so clients open several       |
| HTTP/2   | TCP           | Many concurrent streams over one connection                     |
| HTTP/3   | QUIC over UDP | Many streams, and a lost packet delays only its own stream      |

Application code rarely changes between versions, because handlers see the same methods, headers and bodies. The differences show up in performance. On HTTP/2, one lost TCP packet delays every stream on that connection until it is retransmitted, which is the problem HTTP/3 was designed to remove.

Connections are normally encrypted with [TLS](./TLS.md) and reused through [keep-alive](./Keep-alive.md). HTTP defines the message format but not the design of your API. Conventions such as [REST](./REST.md) sit on top of it, and [gRPC](./gRPC.md) uses HTTP/2 as its transport.

_Usage:_

"The endpoint works in Chrome but curl gets a 401."

"HTTP is stateless, so curl knows nothing about your login. Copy the cookie header across and try again."

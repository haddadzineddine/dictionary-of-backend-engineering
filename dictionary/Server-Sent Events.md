---
description: A one-way stream of text events from server to client over a single long-lived HTTP response, with built-in browser reconnection.
---

Server-Sent Events (SSE) is a standard for streaming messages from a server to a client over one long-running [HTTP](./HTTP.md) response. The server replies with `Content-Type: text/event-stream`, leaves the response open and writes events as they happen:

```text
id: 42
event: order-updated
data: {"orderId": 7, "status": "shipped"}

```

A blank line ends each event. In browsers, the built-in `EventSource` API parses the stream and reconnects automatically after a drop. On reconnect it sends the last ID it received in a `Last-Event-ID` header, so the server can replay what was missed if it has kept those events.

SSE fits notifications, progress bars, live dashboards and streamed text. Because it is plain HTTP, it works with existing [middleware](./Middleware.md), cookies and routing. Client-to-server messages travel as normal requests. When both directions are frequent, a [WebSocket](./WebSocket.md) is a better fit.

The most common problem is buffering. A proxy or compression layer may hold output until it has a full chunk, so events that were sent one at a time arrive in a late batch, or only when the stream ends. Disable buffering and compression for this route at each hop. With nginx, for example, the response header `X-Accel-Buffering: no` turns off proxy buffering.

Other limits to plan for:

| Limit                 | Effect                                                     | Response                              |
| --------------------- | ---------------------------------------------------------- | ------------------------------------- |
| HTTP/1.1 browser cap  | About six connections per origin, shared across tabs       | Serve over HTTP/2                     |
| Idle timeouts         | Quiet streams get closed                                   | Send a comment line such as `: ping`  |
| No custom headers     | `EventSource` cannot set `Authorization`                   | Use a cookie or a fetch-based client  |
| Open connections      | Each client holds server resources                         | Prefer an [event loop](./Event%20loop.md) server |

_Usage:_

"The progress bar sits at zero, then jumps to 100 at the end."

"The events are being buffered by nginx. Turn buffering off for the SSE route."

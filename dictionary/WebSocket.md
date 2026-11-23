---
description: A long-lived, two-way connection between client and server, started from HTTP, for messages flowing in both directions.
---

A WebSocket is a persistent connection over which client and server can each send messages at any time. It suits features where both sides talk frequently, such as chat, collaborative editing and multiplayer state.

The connection starts as an ordinary [HTTP](./HTTP.md) request carrying an `Upgrade: websocket` header. The server answers with [status code](./Status%20code.md) `101 Switching Protocols`, and from then on the same connection carries WebSocket frames instead of HTTP messages. Use the encrypted `wss://` scheme, which runs over [TLS](./TLS.md).

Long-lived connections change how a server behaves in production:

| Problem                  | Symptom                                     | What to do                                       |
| ------------------------ | ------------------------------------------- | ------------------------------------------------ |
| Idle timeout at a proxy  | Sockets drop after a fixed quiet period     | Send ping/pong heartbeats                        |
| Half-open connections    | Server keeps "online" users who have left   | Close sockets that miss heartbeats               |
| Deploys                  | Every client disconnects at once            | Reconnect with random backoff, drain slowly      |
| Several server instances | A message misses users on another instance  | Publish through a shared message broker          |
| Slow consumers           | Server memory grows                         | Limit buffers and disconnect slow clients        |

Clients must assume the connection will drop, and must be able to catch up on anything missed after reconnecting. Without random delay, a deploy causes all clients to reconnect at the same instant.

Security needs specific attention. Browsers attach cookies to the handshake, and WebSockets are not restricted by the same-origin policy, so a malicious page can open a socket as the logged-in user. Validate the `Origin` header during the handshake, perform [authentication](./Authentication.md) there, and check [authorization](./Authorization.md) on each message.

If data flows only from server to client, [Server-Sent Events](./Server-Sent%20Events.md) are simpler to run.

_Usage:_

"Everyone's live cursor freezes after about a minute of inactivity."

"The proxy is closing idle WebSocket connections. Add a heartbeat every 30 seconds and reconnect on close."

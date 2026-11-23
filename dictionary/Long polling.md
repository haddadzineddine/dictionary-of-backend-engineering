---
description: A client request the server holds open until data is ready or a timeout passes, after which the client immediately asks again.
---

Long polling is a technique for receiving updates over ordinary [HTTP](./HTTP.md) requests. The client sends a request, and the server holds it open instead of answering immediately. When new data appears, the server responds. If nothing happens before a timeout, it returns an empty response such as `204 No Content`. Either way, the client immediately sends the next request.

Regular polling asks on a fixed schedule, so most responses are empty and updates wait for the next tick. Long polling delivers an update soon after it occurs and needs no special protocol, so it works through most proxies and firewalls.

| Technique | Direction | Latency | Connection cost | Infrastructure |
| --------- | --------- | ------- | --------------- | -------------- |
| Short polling | Client asks | Up to one interval | Repeated requests | Plain HTTP |
| Long polling | Server to client | Low | One held request per client | Plain HTTP |
| [Server-Sent Events](./Server-Sent%20Events.md) | Server to client | Low | One open stream | Plain HTTP, no buffering |
| [WebSocket](./WebSocket.md) | Both | Low | One open socket | Upgrade support |

A common bug is losing events between polls. If an event fires after one response is sent and before the next request arrives, a server that only reports "what happens while you are connected" never delivers it. Have the client send the ID or cursor of the last event it received, and return everything after that point.

Set the hold time below every timeout in the path. If a proxy gives up at 30 seconds and the server holds for 60, clients see regular 504 errors during quiet periods. Add a short random delay after errors so a restart does not bring every client back at once.

Held requests also consume capacity. On a server that dedicates a [thread](./Thread.md) to each request, a few hundred waiting clients can exhaust the pool, so this pattern fits non-blocking servers better.

_Usage:_

"Users sometimes miss a notification until they refresh."

"Our long polling drops anything that fires between requests. Send the last event ID on each poll."

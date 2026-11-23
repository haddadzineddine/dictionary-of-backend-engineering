---
description: Stopping a server by refusing new work, finishing what is in flight and closing resources before the process exits.
---

Graceful shutdown is stopping a process in an orderly way: it refuses new work, finishes what is in progress, closes its resources and exits. Without it, each deploy kills in-flight requests, which appears as a burst of 502 errors at every release.

Platforms normally signal a process with `SIGTERM`, wait for a grace period, then force it to stop with `SIGKILL`, which cannot be caught. Kubernetes, for example, waits 30 seconds by default. A server should then:

| Step | Who     | What happens                                                                 |
| ---- | ------- | ---------------------------------------------------------------------------- |
| 1    | Process | Receives `SIGTERM`                                                           |
| 2    | Process | Fails its readiness check so traffic stops arriving       |
| 3    | Process | Waits briefly, because routing updates are not instant                       |
| 4    | Process | Stops accepting connections and closes idle [keep-alive](./Keep-alive.md) ones |
| 5    | Process | Lets in-flight requests finish, up to a deadline                             |
| 6    | Process | Closes database pools and flushes logs and metrics                           |
| 7    | Process | Exits before the grace period ends                                           |

Step 3 is often missed. If the server closes its listener immediately, the load balancer (the component spreading traffic across instances) can still route requests to it for a few seconds, and those requests fail.

A [WebSocket](./WebSocket.md) or [Server-Sent Events](./Server-Sent%20Events.md) stream never finishes alone, so the server must close it and rely on clients reconnecting with random delay.

A [worker](./Worker.md) stops reserving jobs, then finishes the current job or returns it to the [job queue](./Job%20queue.md). If a job runs longer than the grace period, it will be killed, so it must be safe to retry.

A common container bug blocks this. If the image starts the app through a shell, as in `CMD node server.js`, the shell may receive `SIGTERM` and not forward it. The app is then killed at the deadline. Use the exec form, `CMD ["node", "server.js"]`.

_Usage:_

"We get a spike of 502s on every deploy."

"The app has no graceful shutdown. Handle `SIGTERM`, fail readiness, then drain before exiting."

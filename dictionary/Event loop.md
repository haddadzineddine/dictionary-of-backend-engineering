---
description: A loop that waits for ready events and runs their callbacks on one thread, letting it serve many connections that mostly wait.
---

An event loop is a loop that waits for events, such as data arriving on a socket, and runs the code registered for each one. It lets a single [thread](./Thread.md) serve thousands of connections, because a connection that is waiting for data costs no thread. Node.js, Python's asyncio, Nginx and Redis are built around this design.

| Step | Who         | What happens                                                  |
| ---- | ----------- | ------------------------------------------------------------- |
| 1    | Application | Starts an I/O operation and registers what to do on completion |
| 2    | Event loop  | Continues with other ready work                               |
| 3    | OS          | Reports ready sockets through a mechanism such as epoll or kqueue |
| 4    | Event loop  | Queues the matching callback or resumes the paused task       |
| 5    | Event loop  | Runs it until it finishes or yields, then repeats             |

Scheduling is cooperative: nothing interrupts running code. A task keeps the thread until it returns or reaches an `await` in [async-await](./Async-await.md) code. The main failure follows from this: one slow operation freezes every other request in the process. A 2-second `JSON.parse` on a large payload, a synchronous file read, an expensive regular expression or [password hashing](./Password%20hashing.md) on the main thread means all users wait 2 seconds.

The symptom is latency rising across all endpoints at once, with one CPU core at 100%. Measure event loop lag, the delay between when a callback was due and when it ran.

To keep the loop responsive, avoid [blocking I/O](./Blocking%20IO.md) on it. Move CPU-heavy work to a separate thread, a separate process or a [background job](./Background%20job.md). Limit request body sizes and split long computations.

"Single-threaded" describes the part that runs your code. Node.js also uses a small internal [thread pool](./Thread%20pool.md) for some file, DNS lookup and crypto operations. It has four threads by default, so those operations can queue behind each other.

_Usage:_

"Every endpoint spikes whenever someone downloads the big report."

"The report builder runs on the event loop and blocks it. Move it off the main thread."

---
description: An input or output call that stops the calling thread until it completes, leaving that thread unable to do anything else.
---

Blocking I/O is an input or output operation that suspends the calling [thread](./Thread.md) until it completes. A call such as `read()` on a socket does not return until data arrives. While it waits, the thread uses almost no CPU, but it cannot do anything else. Network calls, disk access and database queries are the usual examples.

Non-blocking I/O returns control immediately, and the program learns later that the operation has finished.

| Aspect              | Blocking                         | Non-blocking                               |
| ------------------- | -------------------------------- | ------------------------------------------ |
| Code style          | Sequential and easy to read      | [Async-await](./Async-await.md), callbacks |
| Waiting costs       | One thread per in-flight call    | A small record per in-flight call          |
| Typical server      | [Thread pool](./Thread%20pool.md) | [Event loop](./Event%20loop.md)           |
| Main risk           | Running out of threads           | One blocking call stalls everyone          |

Blocking is fine in a threaded server, provided the pool is sized for it and every call has a timeout. Without a timeout, a dependency that stops answering holds its thread indefinitely, and enough of those calls will exhaust the pool.

The damaging case is a blocking call inside an event loop. In Node.js, `fs.readFileSync` stops every request in the process while it runs. In Python, calling `time.sleep()` or the synchronous `requests` library inside an `async def` handler freezes the loop in the same way. The code looks correct and passes tests with one user, then throughput collapses under load because requests are handled one at a time.

To fix it, use the async version of the library, or move the call to a separate thread, for example with `asyncio.to_thread` in Python.

Not all I/O has a non-blocking form on every platform. Regular file reads on Linux generally block even when a descriptor is marked non-blocking, so runtimes such as Node.js hand file operations to helper threads.

_Usage:_

"Why does one slow upload stall the whole FastAPI app?"

"That `async def` handler does blocking I/O with `requests`. Use an async client or run it in a thread."

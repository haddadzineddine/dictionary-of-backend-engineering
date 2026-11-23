---
description: Language syntax for writing non-blocking code in sequential style, pausing a function at each await until its result is ready.
---

Async-await is language syntax for writing non-blocking code that reads like sequential code. An `async` function can pause at each `await`, handing control back to the scheduler, and resume when the awaited operation completes. JavaScript, Python, C# and Rust all provide it. Usually an [event loop](./Event%20loop.md) or similar runtime does the scheduling.

```javascript
async function getOrder(id) {
  const order = await db.findOrder(id); // other tasks run during this wait
  const user = await db.findUser(order.userId);
  return { order, user };
}
```

The compiler or runtime turns such a function into a resumable state machine, so a paused function holds no [thread](./Thread.md). This gives [concurrency](./Concurrency.md) for I/O-bound work. It does not make CPU-heavy code faster or run it in [parallel](./Parallelism.md).

Common mistakes have recognisable symptoms:

| Mistake                         | Symptom                                                 | Fix                                       |
| ------------------------------- | ------------------------------------------------------- | ----------------------------------------- |
| Forgotten `await`               | Code continues early, errors vanish or go unhandled     | Enable a lint rule for floating promises  |
| Awaiting in a loop              | 20 independent calls run one after another              | `Promise.all`, `asyncio.gather`           |
| Unbounded `Promise.all`         | 10,000 calls at once exhaust connections                | Add a concurrency limit                   |
| [Blocking I/O](./Blocking%20IO.md) in an async function | Every request stalls                 | Use an async client or a separate thread  |
| Fire-and-forget task            | Work is lost on restart, failures go unseen             | Use a [background job](./Background%20job.md) |

A common misconception is that single-threaded async code cannot have a [race condition](./Race%20condition.md). It can. Other tasks run at every `await`, so a sequence that reads a balance, awaits something, then writes the balance can interleave with another request doing the same. Keep check-and-update logic free of awaits, or make it atomic in the database.

Async also spreads through a codebase: a function that awaits must generally be async itself, and so must its callers. This is why mixing sync and async libraries takes planning.

_Usage:_

"The endpoint makes five API calls and takes five seconds."

"You're using async-await one call at a time. They're independent, so run them with `Promise.all`."

---
description: A bug where the result depends on the timing of overlapping operations, so it appears only under particular interleavings.
---

A race condition is a bug where the outcome depends on the timing of overlapping operations. The code is correct when steps run one after another and wrong when they interleave. It explains failures such as an oversold last ticket, a coupon redeemed twice, or a counter that ends up lower than the number of requests. These bugs often pass tests and appear only under production load, because [concurrency](./Concurrency.md) is what exposes them.

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
| Lock                    | A [mutex](./Mutex.md) or a row lock around the critical steps |
| Remove sharing          | Process each account's work through one ordered queue        |

An in-process mutex protects only that process. With several server instances, two requests can hold "the" lock at the same time on different machines, so the rule must be enforced in a shared system such as the database.

Single-threaded runtimes are not immune, because tasks interleave at each `await` in [async-await](./Async-await.md) code.

_Usage:_

"We sold 102 tickets for a 100-seat venue."

"That's a race condition in the availability check. Make the decrement atomic and conditional on remaining seats."

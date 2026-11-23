---
description: A test that passes and fails on the same code with no change, usually because of timing, shared state or environment.
---

A flaky test is a test that sometimes passes and sometimes fails on identical code. It often appears as a test that passes alone and fails in CI, the automated build, or a pipeline that goes green when someone presses "re-run". The damage spreads beyond the single test: once people expect random failures, they start ignoring real ones.

Most flakiness has a small set of causes:

| Cause            | Example                                                | Fix                                         |
| ---------------- | ------------------------------------------------------ | ------------------------------------------- |
| Timing           | `sleep(1)` before checking a [background job](./Background%20job.md) | Poll for the condition with a timeout |
| Shared state     | Tests reuse one database row or global variable        | Isolate data per test                       |
| Test order       | One test depends on another's leftovers                | Run in random order to expose it            |
| Clock            | Fails at midnight, month-end or in another timezone    | Inject a fixed clock                        |
| Unordered results | Asserting on a query with no `ORDER BY`               | Sort, or compare as a set                   |
| Network          | Calls a real third-party sandbox                       | Use a [test double](./Test%20double.md) or stub server |
| Code bug         | A [race condition](./Race%20condition.md) in the code  | Fix the production bug                      |

The last row is why flakiness should not be dismissed. A test failing once in fifty runs may be correctly detecting a race that also hits customers.

To reproduce one, run it in a loop, run the suite with the same order seed, and limit CPU to mimic CI.

Automatic retries keep a pipeline moving, but they also hide the problem. If you use them, record every test that passed only on retry. Quarantine a known flaky test by moving it out of the blocking suite, assign an owner and set a deadline.

[Integration tests](./Integration%20test.md) and end-to-end tests are more prone to this, which is one reason the [test pyramid](./Test%20pyramid.md) keeps them fewer.

_Usage:_

"Just re-run it, that test always fails on Fridays."

"That's a flaky test with a date bug. Inject the clock and fix it instead of re-running."

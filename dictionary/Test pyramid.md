---
description: A model for balancing a test suite, with many fast unit tests, fewer integration tests and a small number of end-to-end tests.
---

The test pyramid is a model for deciding how many tests to write at each level. Mike Cohn popularised it. Its shape reflects cost: the base holds many fast, cheap tests, and each level above holds fewer, slower and more realistic ones.

| Level                                     | Quantity | Speed    | Confidence given             | Failure points to     |
| ----------------------------------------- | -------- | -------- | ---------------------------- | --------------------- |
| End-to-end                                | Few      | Slow     | The full system works        | Anywhere              |
| [Integration](./Integration%20test.md)    | Some     | Medium   | Components work together     | A specific boundary   |
| [Unit](./Unit%20test.md)                  | Many     | Fast     | Logic is correct             | A small area of code  |

An end-to-end test drives the deployed system as a user or client would. It catches wiring problems nothing else can, but it is slow, can fail for unrelated reasons, and a failure reveals little about the cause.

The usual rule is to test each behaviour at the lowest level that can catch its bug. Cover all discount permutations in unit tests. Use one integration test to show that the discount persists, and one end-to-end test to show that checkout works.

The recognisable anti-pattern is the inverted pyramid, or "ice cream cone". Most coverage sits in browser or end-to-end tests. CI, the automated build, takes an hour, a [flaky test](./Flaky%20test.md) fails most runs, and developers rerun the pipeline until it passes or stop reading the results. When a test fails at the top and no lower test failed with it, consider adding the missing lower-level test.

The pyramid is a guideline and not a fixed ratio. A thin API over a database has little isolated logic, so integration tests may provide most of the value. That view is reflected in alternatives such as the "testing trophy". A [contract test](./Contract%20test.md) can replace many slow cross-service end-to-end tests.

_Usage:_

"CI takes 70 minutes, and most of that is Selenium."

"Our test pyramid is upside down. Keep a few critical journeys and push the rest down to API and unit tests."

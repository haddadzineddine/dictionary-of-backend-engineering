---
description: A fast, isolated test of one small piece of behaviour, run without a network, real database or other external system.
---

A unit test is an automated test that checks a small piece of behaviour in isolation. It runs in memory, with no network or real database, so a suite of thousands can finish in seconds and give feedback while you code. Teams disagree on the size of a "unit": it may be one function, one class or a small group of collaborating objects.

```python
def test_discount_is_capped_at_fifty_percent():
    order = Order(subtotal=Money(100))        # arrange
    order.apply_discount(percent=80)          # act
    assert order.total == Money(50)           # assert
```

Unit tests suit logic with many cases: pricing, validation, parsing, date handling and state transitions. Boundary values such as zero, empty input, the maximum and the day of a leap year are cheap to cover here and expensive to cover any other way.

Good unit tests share a few properties:

| Property      | What it means                                      |
| ------------- | -------------------------------------------------- |
| Fast          | Milliseconds each, so people run them constantly   |
| Deterministic | Same result every run, with clock and randomness controlled |
| Isolated      | No dependence on other tests or their order        |
| Behavioural   | Asserts outcomes, not internal steps               |
| Readable      | The name states the rule being protected           |

The common failure is coupling tests to implementation. A test asserting that a private helper was called twice breaks when you refactor, even though the behaviour is unchanged. After enough false alarms, people stop trusting the suite and refactoring slows down. Overuse of a mocking [test double](./Test%20double.md) is the usual cause. Prefer checking return values and resulting state, and use doubles mainly for slow or external dependencies.

Unit tests cannot show that components work together. A query can be mocked successfully and still contain invalid SQL, which is what an [integration test](./Integration%20test.md) is for. The [test pyramid](./Test%20pyramid.md) describes how to balance these levels. Code that is difficult to unit test often has hard-wired dependencies, which [dependency injection](./Dependency%20injection.md) can loosen.

_Usage:_

"I renamed one private method and 40 unit tests failed."

"They're asserting on internals. Rewrite them to check the output, so a refactor doesn't break them."

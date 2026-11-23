---
description: A stand-in for a real dependency during a test. Dummies, stubs, spies, mocks and fakes are the main kinds.
---

A test double is an object that replaces a real dependency during a test, in the way a stunt double replaces an actor. Gerard Meszaros coined the term in his book _xUnit Test Patterns_. Doubles let a [unit test](./Unit%20test.md) avoid sending real email, charging a real card or waiting on a network, and they make hard-to-trigger cases, such as a timeout, easy to simulate.

"Mock" is commonly used for all of them, but there are five distinct kinds:

| Kind  | What it does                                  | Example                                  |
| ----- | --------------------------------------------- | ---------------------------------------- |
| Dummy | Fills a parameter and is never used           | A placeholder logger                     |
| Stub  | Returns canned answers                        | `getRate()` always returns `1.2`         |
| Spy   | Records how it was called                     | Captures emails "sent" for later checks  |
| Mock  | Is preprogrammed with expected calls and verifies them | Fails unless `charge(100)` is called once |
| Fake  | Has a working but simplified implementation   | An in-memory [repository](./Repository%20pattern.md) |

Choosing the right kind affects how brittle a suite becomes. Stubs and fakes support checking outcomes. Mocks verify interactions, which fits cases where the call itself is the behaviour, such as "the payment was charged exactly once". Using mocks for every collaborator ties tests to implementation, so a harmless refactor turns the suite red.

The larger risk is drift. A double encodes your belief about how the dependency behaves, and it keeps passing after the real one changes. The stub returns `{ "amount": 100 }` while the real API now returns a string, so tests stay green and production fails. An [integration test](./Integration%20test.md) or [contract test](./Contract%20test.md) keeps those beliefs honest.

Double what you own, by wrapping a third-party SDK in a small interface of your own and replacing that interface. Inject doubles through [dependency injection](./Dependency%20injection.md) instead of patching globals, which can leak between tests and cause a [flaky test](./Flaky%20test.md).

_Usage:_

"The payment test passes, but real charges are failing."

"Our test double still returns the old response format. Add a contract test against the provider."

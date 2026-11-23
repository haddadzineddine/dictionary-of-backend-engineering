---
description: A test that runs your code against real collaborators, such as a database, to check that the parts work together.
---

An integration test checks that your code works correctly with a real collaborator, such as a database, message broker or another module. A [unit test](./Unit%20test.md) replaces those dependencies, so it cannot catch a mistaken assumption about how they behave. Integration tests catch failures that appear at the join.

The familiar symptom is a green test suite followed by a production failure. The mocked [repository](./Repository%20pattern.md) returned a row, but the real query had a typo. The code inserted a duplicate that the real unique constraint rejected. A timestamp lost precision, a migration was missing, or JSON serialized differently from what the test assumed.

Good targets include:

| Target                                     | What it can catch                                  |
| ------------------------------------------ | -------------------------------------------------- |
| Repository and [ORM](./ORM.md) queries | Invalid SQL, constraints, N+1, collation |
| [HTTP](./HTTP.md) route through [middleware](./Middleware.md) | Routing, [authentication](./Authentication.md), [status codes](./Status%20code.md) |
| [Job queue](./Job%20queue.md) flows        | Serialization, retries, acknowledgement            |
| Transaction behaviour                      | Rollbacks and isolation between operations         |
| Client for an external API                 | Parsing, timeouts and error mapping, using a local stub server |

Use the same database engine as production. Running tests on SQLite while production uses PostgreSQL hides differences in types, locking and SQL dialect. Tools such as Testcontainers start a real database in a container, disposable per test run.

Isolation is the main practical challenge. Tests sharing a database can interfere and produce a [flaky test](./Flaky%20test.md) that only fails when run in a certain order or in parallel. Common solutions include wrapping each test in a transaction that rolls back, giving each test process its own schema, or generating unique data per test.

Integration tests are slower than unit tests, so keep them focused on the boundary and leave detailed business-rule permutations to faster tests. For a dependency owned by another team, a [contract test](./Contract%20test.md) is often more practical than running their full system.

_Usage:_

"All the tests pass, but the deploy fails on a unique constraint."

"The repository is mocked everywhere. Add an integration test against a real Postgres container."

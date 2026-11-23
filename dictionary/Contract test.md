---
description: A test verifying that an API provider and its consumer agree on request and response format, without running both together.
---

A contract test verifies that two services agree on the format of their communication, without deploying them together. The contract records the requests a consumer sends and the responses it depends on. Each side then tests against that record independently.

It addresses a specific gap. Team A's tests mock Team B's API and pass. Team B's tests pass too. Then B renames a field, both pipelines stay green, and the failure appears only after deployment. Each side was testing against its own assumption. A contract test turns that mismatch into a failing build before the [breaking change](./Breaking%20change.md) ships.

In consumer-driven contract testing, the flow is as follows:

| Step | Who      | What happens                                                           |
| ---- | -------- | ---------------------------------------------------------------------- |
| 1    | Consumer | Writes a test declaring the request it sends and the fields it needs   |
| 2    | Tool     | Runs that test against a mock and generates a contract file            |
| 3    | Consumer | Publishes the contract to a shared location                            |
| 4    | Provider | Replays the contract against its real code in CI, the automated build  |
| 5    | Both     | Check compatibility before deploying                                   |

Pact is a widely used tool for this approach. When every consumer publishes a contract, a provider can see which fields are unused.

An alternative is schema-based testing, where both sides are checked against a shared [OpenAPI](./OpenAPI.md) document or [Protocol Buffers](./Protocol%20Buffers.md) definition. This is simpler to adopt and suits public APIs with unknown consumers, but it shows only that a schema is compatible, and it says nothing about which fields clients rely on.

Contract tests check structure and agreed interactions. They do not verify business logic, so a response with the correct shape and a wrong price still passes. Keep those checks in the provider's [unit tests](./Unit%20test.md) and [integration tests](./Integration%20test.md).

_Usage:_

"Their deploy broke our checkout again, and both CI pipelines were green."

"We're mocking their API from memory. Add a contract test so their build fails if they drop a field we use."

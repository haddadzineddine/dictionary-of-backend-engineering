---
description: A boundary within which a domain model and its terms have one consistent meaning, even if other areas use the words differently.
---

A bounded context is an explicit boundary inside which a model and its vocabulary have one consistent meaning. The term comes from [domain-driven design](./Domain-driven%20design.md). It accepts that a large business does not have one universal model, because the same word means different things to different departments.

| Context  | What "customer" means               | Fields it cares about           |
| -------- | ----------------------------------- | ------------------------------- |
| Sales    | A lead or account being pursued     | Stage, owner, expected value    |
| Billing  | The party that pays invoices        | Tax ID, payment method, balance |
| Shipping | A recipient at an address           | Address, delivery instructions  |
| Support  | A person with open tickets          | Plan, ticket history            |

Without boundaries, these meanings merge into a single model. The symptom is a `User` or `Customer` class with a hundred fields and many nullable columns, which every team edits and nobody fully understands. A billing change breaks shipping, and developers argue over what "active" means because each team is correct about its own area.

Inside each bounded context, the model contains only what that context needs. Billing has its own `Customer`, shipping has its own `Recipient`, and they relate through a shared identifier. Some duplication is intended, since it allows each model to change independently.

Contexts communicate through explicit contracts, such as an API, a [webhook](./Webhook.md) or published events. When one integrates with a messy or external model, it can use an anti-corruption layer, code that translates the foreign model at the edge so its concepts do not leak inward. A [DTO](./DTO.md) often carries the data across this boundary.

A bounded context is a design boundary and does not have to be a separate service. It can be a module inside one deployable application, which is often a sensible starting point. Enforce it by not reaching into another context's tables or internal classes.

_Usage:_

"Should we add `shipping_notes` to the main `Customer` model?"

"No, that belongs to the shipping bounded context. Keep it on `Recipient` and link by customer ID."

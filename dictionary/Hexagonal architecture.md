---
description: Structuring an app so business logic sits at the centre and reaches databases, APIs and UIs only through interfaces called ports.
---

Hexagonal architecture, also called ports and adapters, is a structure that keeps business logic independent of technical details. Alistair Cockburn proposed it. The application sits at the centre and communicates with the outside world only through interfaces called ports. Adapters implement those ports for specific technologies.

| Part            | What it is                                  | Example                            |
| --------------- | ------------------------------------------- | ---------------------------------- |
| Application     | Use cases and business rules                | `PlaceOrder`                       |
| Inbound port    | How the outside calls the application       | `PlaceOrder.execute(command)`      |
| Inbound adapter | Translates an external trigger into a call  | [HTTP](./HTTP.md) controller, queue consumer |
| Outbound port   | What the application needs from outside     | `PaymentGateway` interface         |
| Outbound adapter | Implements that need with a technology     | Stripe client, Postgres [repository](./Repository%20pattern.md) |

The defining rule is that dependencies point inward. The application defines `PaymentGateway`, and the payment adapter implements it. This differs from traditional [layered architecture](./Layered%20architecture.md), where business code usually depends directly on the data layer. Adapters are connected at startup, normally through [dependency injection](./Dependency%20injection.md).

The structure addresses a familiar problem: business logic that cannot be tested without a running database and a sandbox payment account, or a vendor's SDK types spread across hundreds of files so that changing vendors becomes a rewrite. With ports, a [unit test](./Unit%20test.md) can run a use case with a [test double](./Test%20double.md) for each outbound port. The same use case can be triggered by HTTP, a [cron job](./Cron%20job.md) or a [worker](./Worker.md) by adding an adapter.

The cost is indirection. Each external interaction needs an interface, an adapter and mapping code, which is unnecessary for a thin CRUD application with few rules. A port that mirrors a vendor's API keeps the coupling it was supposed to remove, so define ports in terms of what the application needs. Adapters still need an [integration test](./Integration%20test.md), since test doubles cannot show that the real technology behaves as assumed.

_Usage:_

"Replacing the payment provider means touching 200 files."

"Because the SDK leaked everywhere. With hexagonal architecture, we'd only write a new adapter for the payment port."

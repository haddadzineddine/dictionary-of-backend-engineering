---
description: Hiding data access behind a collection-like interface so business code asks for domain objects without knowing about queries.
---

The repository pattern places data access behind an interface that resembles a collection of domain objects. Business code calls methods such as `orders.findById(id)` or `orders.save(order)` and does not need to know how the data is stored. The pattern was described in Martin Fowler's catalogue of enterprise patterns and is a building block of [domain-driven design](./Domain-driven%20design.md).

```typescript
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findOverdue(asOf: Date): Promise<Order[]>;
  save(order: Order): Promise<void>;
}
```

It addresses query logic scattered across a codebase. When the definition of an "active customer" appears in fifteen handwritten queries, a change to the rule leaves some of them behind, and reports begin to disagree. A named method such as `findActive()` puts the definition in one place. The interface also lets a [unit test](./Unit%20test.md) use an in-memory [test double](./Test%20double.md), and it serves as an outbound port in [hexagonal architecture](./Hexagonal%20architecture.md).

The pattern is often misapplied:

| Misuse                          | Result                                                  | Better approach                       |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------- |
| Generic `Repository<T>` for all | A thin wrapper repeating the [ORM](./ORM.md)            | Methods named for real queries        |
| Returning query builders        | Storage details leak to callers                         | Return domain objects                 |
| Loops calling `findById`        | One query per item, slow lists                          | Add a batch method                    |
| One method per screen           | The interface grows without limit                       | Use separate read queries             |
| Each repository commits alone   | A multi-step operation can half-succeed                 | Manage the transaction at the use case |

Transactions need attention. If `orders.save()` and `inventory.save()` commit separately, a failure between them leaves inconsistent data. Have the application layer open one transaction, sometimes through a unit-of-work object, that both repositories join.

An in-memory fake will not reproduce real constraints, collation or locking, so test the actual repository with an [integration test](./Integration%20test.md). For small applications where an ORM already provides a clear interface, adding repositories may only add a layer.

_Usage:_

"Finance and support are showing different counts of active customers."

"Each has its own query. Put the rule in one repository method and use it in both."

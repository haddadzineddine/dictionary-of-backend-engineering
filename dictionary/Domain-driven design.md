---
description: An approach that shapes software around a model of the business, built with domain experts and using their language in code.
---

Domain-driven design (DDD) is an approach to software that shapes code around a shared model of the business problem. Eric Evans introduced it in his 2003 book of the same name. Developers work with domain experts, such as accountants or logistics planners, so that the concepts in the code match how the business operates.

It targets a recognisable problem. The business says "a policy lapses", the code says `status = 3`, and every requirement has to be translated in someone's head. DDD answers this with a ubiquitous language: an agreed vocabulary used in conversation, documents, class names and method names, such as `policy.lapse()`.

DDD has two levels. The strategic level divides a large domain into parts, each with a [bounded context](./Bounded%20context.md) in which terms have one precise meaning. The tactical level gives patterns for modelling inside a context:

| Building block | Meaning                                              | Example                  |
| -------------- | ---------------------------------------------------- | ------------------------ |
| Entity         | Has an identity that persists as data changes        | `Order #42`              |
| Value object   | Defined by its values, usually immutable             | `Money(10, "EUR")`       |
| Aggregate      | A cluster changed as a unit through one root         | `Order` and its lines    |
| Domain event   | A record of something that happened                  | `OrderShipped`           |
| [Repository](./Repository%20pattern.md) | Loads and saves aggregates | `OrderRepository` |

Aggregates protect business rules. If an order total must equal the sum of its lines, all changes go through the `Order` root, so no code can update one line and leave the total wrong.

DDD has a real cost and suits complex, long-lived business domains. A simple CRUD application gains little from aggregates and domain services. A common failure is adopting the tactical patterns without talking to domain experts, which adds classes without improving the model. It pairs well with [hexagonal architecture](./Hexagonal%20architecture.md), which keeps the model free of infrastructure.

_Usage:_

"The code calls it `status 3`, but operations calls it 'on hold pending review'."

"Domain-driven design would have us use their term in the code, so we stop translating."

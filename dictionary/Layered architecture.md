---
description: Organising an application into horizontal layers, such as API, service and data access, where each depends on the one below.
---

Layered architecture is a way of organising an application into horizontal layers, each with one kind of responsibility. Code in a layer calls the layer beneath it and knows nothing about the layers above. It is the default structure of many web frameworks.

| Layer        | Responsibility                               | Typical contents             |
| ------------ | -------------------------------------------- | ---------------------------- |
| Presentation | Handle [HTTP](./HTTP.md), parse input, format output | Controllers, [DTOs](./DTO.md) |
| Application  | Coordinate a use case and its transaction    | Services                     |
| Domain       | Hold business rules                          | Entities, calculations       |
| Data access  | Read and write storage                       | [Repositories](./Repository%20pattern.md), [ORM](./ORM.md) models |

The purpose is to separate reasons for change. A new response format should affect the presentation layer, and a query optimization should affect data access. The structure also gives a team a shared answer to "where does this code go?"

Layering breaks down in recognisable ways. SQL appears in a controller, so the same rule gets reimplemented in three endpoints with slight differences. Business logic lands in controllers, so a [background job](./Background%20job.md) that needs the same rule has to copy it, because it has no HTTP request to pass through. Keeping rules below the presentation layer lets every entry point reuse them.

The opposite failure is ceremony. A service method that only calls a repository method, which only calls the ORM, adds files without adding behaviour. Adding one field then means editing every layer. For a small CRUD service (one that mainly creates, reads, updates and deletes records), fewer layers are reasonable.

A further limitation is the direction of dependencies. In the traditional form, business logic depends on the data layer, so database details can leak upward and the rules are hard to test alone. [Hexagonal architecture](./Hexagonal%20architecture.md) inverts that dependency using interfaces and [dependency injection](./Dependency%20injection.md).

_Usage:_

"Where should the discount calculation go? It's in the controller right now."

"Move it down. In our layered architecture, the [worker](./Worker.md) and the API should both call the same domain code."

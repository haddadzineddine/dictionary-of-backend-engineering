---
description: A library that maps database tables to objects in code and generates SQL for reading and writing them.
---

An ORM (object-relational mapper) is a library that maps database tables to classes and rows to objects, and generates the SQL to read and write them. Hibernate, Entity Framework, SQLAlchemy and Active Record are common examples. Instead of writing an `INSERT`, you create an object and save it.

An ORM removes repetitive query and mapping code, and its standard query methods use bound parameters, which protects them from [SQL injection](./SQL%20injection.md). Many also track changes to loaded objects and write them in one transaction.

The trade-off is that database work becomes less visible. A line that looks like property access can run a query, and this leads to the failures most ORM users eventually meet:

| Problem           | Symptom                                               | What to do                                |
| ----------------- | ----------------------------------------------------- | ----------------------------------------- |
| N+1 queries       | A page of 50 orders runs 51 queries                   | Eager-load the relation or batch it       |
| Over-fetching     | A list loads large text columns it never shows        | Select only needed columns                |
| Lazy load after close | Error when serializing outside the session        | Load data before leaving the transaction  |
| Bulk updates      | Hooks and validations are skipped                     | Know which APIs bypass model logic        |

N+1 usually passes in development with three rows, then becomes a slow endpoint in production. Turn on SQL logging locally, and consider a test that asserts the number of queries for important endpoints.

Use raw SQL when the ORM gets in the way, such as for reports, window functions or bulk changes. Mixing the two is normal. Raw queries must still use bound parameters.

Avoid passing ORM entities through the entire application. Exposing them from an API couples your contract to your table structure, which is what a [DTO](./DTO.md) prevents. A [repository](./Repository%20pattern.md) can keep ORM details out of business logic.

_Usage:_

"This endpoint is fine locally but takes four seconds in production."

"Turn on query logging. I'd bet the ORM is lazy-loading the customer for each order."

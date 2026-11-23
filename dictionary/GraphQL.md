---
description: A query language and API runtime where clients request exactly the fields they need from a typed schema, usually via one endpoint.
---

GraphQL is a query language and server runtime for APIs. The server publishes a typed schema describing its data, and each client sends a query naming the fields it wants, usually to a single endpoint over [HTTP](./HTTP.md). The response has the same shape as the query.

```graphql
query {
  order(id: "42") {
    status
    customer { name }
  }
}
```

This addresses two problems common with [REST](./REST.md): fetching more data than a screen needs, and making several round trips to assemble one view. The server implements a resolver, a small function that fetches one field.

That flexibility moves work and risk to the server:

| Problem            | Symptom                                               | What to do                                   |
| ------------------ | ----------------------------------------------------- | -------------------------------------------- |
| N+1 queries        | A list of 50 items triggers 51 database queries       | Batch lookups per request, as DataLoader does |
| Expensive queries  | A deeply nested query ties up the server              | Enforce depth and cost limits, plus timeouts |
| Field-level access | A private field is reachable through another type     | Apply [authorization](./Authorization.md) in the business layer |
| HTTP caching       | Queries sent by POST bypass shared caches             | Use persisted queries sent by GET            |
| Error handling     | Failures hidden behind a 200                          | Monitor the `errors` array in responses      |

The access-control row deserves care. Because the graph can reach the same object along several paths, a check placed on one resolver can be bypassed by another path to the same data. Put the rule in code that every path shares.

Public GraphQL APIs accept arbitrary queries from strangers, so limits are required. For APIs used only by your own clients, an allowlist of persisted queries, where only pre-registered queries can run, closes off most of this risk.

Evolving a schema usually means adding fields and marking old ones `@deprecated` instead of using [API versioning](./API%20versioning.md). Removing a field is still a [breaking change](./Breaking%20change.md).

_Usage:_

"The orders page got slow after we added the customer's name."

"That's N+1 in the GraphQL resolver. It fetches each customer separately, so batch them with a loader."

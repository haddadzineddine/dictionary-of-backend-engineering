---
description: Running more than one version of an API contract so existing clients keep working while the interface changes.
---

API versioning is a way of publishing changed behaviour without breaking clients that depend on the old behaviour. Clients state which version they expect, and the server honours that contract. It exists because you cannot update every consumer at the moment you deploy, particularly mobile apps and third-party integrations.

Versioning is needed for a [breaking change](./Breaking%20change.md). Additive changes, such as a new optional field, normally ship without a new version.

| Strategy          | Example                              | Trade-off                                       |
| ----------------- | ------------------------------------ | ----------------------------------------------- |
| URL path          | `/v2/orders`                         | Obvious and easy to route, but coarse           |
| Custom header     | `Api-Version: 2`                     | Clean URLs, harder to test in a browser         |
| Media type        | `Accept: application/vnd.acme.v2+json` | Precise, less familiar to clients             |
| Query parameter   | `/orders?version=2`                  | Easy to use, can complicate caching             |
| Dated versions    | `Api-Version: 2026-01-15`            | Fine-grained, requires a translation layer      |

If a version is selected by a header, any cache in front of the API must vary on that header. Otherwise one client can receive a response cached for another version.

The ongoing cost is maintenance. Each live version has to be tested, patched and kept secure, and copying handlers per version multiplies every future bug fix. A common structure keeps one current implementation and adds thin adapters that translate older requests and responses at the boundary.

A version also needs a retirement plan. Track usage per version and per client, announce a date, and signal it in responses, for example with the standard `Sunset` header. Contact remaining users before the date. Teams that skip this step end up supporting `/v1` indefinitely because nobody knows who still calls it.

[GraphQL](./GraphQL.md) APIs tend to avoid numbered versions by deprecating fields, and [Protocol Buffers](./Protocol%20Buffers.md) schemas evolve through compatible changes, but both still need a plan for removing old fields.

_Usage:_

"Can we finally delete `/v1`?"

"Not yet. API versioning metrics show one partner still calling it, so send them the sunset date first."

---
description: An architectural style for APIs built around resources, standard HTTP methods and stateless requests.
---

REST (Representational State Transfer) is an architectural style for networked APIs, described by Roy Fielding in his 2000 doctoral dissertation. A REST API exposes resources, such as orders or users, at URLs. Clients act on them through standard [HTTP methods](./HTTP%20method.md) and exchange representations of them, usually JSON.

| Operation           | Request               | Typical success                                   |
| ------------------- | --------------------- | ------------------------------------------------- |
| List orders         | `GET /orders`         | 200                                               |
| Read one order      | `GET /orders/42`      | 200                                               |
| Create an order     | `POST /orders`        | 201 with a `Location` header                      |
| Replace an order    | `PUT /orders/42`      | 200 or 204                                        |
| Update some fields  | `PATCH /orders/42`    | 200 or 204                                        |
| Delete an order     | `DELETE /orders/42`   | 204                                               |

Fielding's constraints include a client-server split, stateless requests, cacheable responses, a uniform interface and a layered system. Statelessness means each request carries its own context, so any server instance can handle it. Cacheability means GET responses can be reused by browsers and proxies when their headers allow it.

Most APIs described as REST follow only part of this. The full style includes hypermedia, where responses contain links telling the client what it can do next, and few JSON APIs implement it. In everyday use, "REST" usually means resource-oriented JSON over [HTTP](./HTTP.md).

The practical benefit comes from following the conventions, because clients, proxies and monitoring already understand them. Endpoints such as `POST /getUser` or a `200` response wrapping an error discard that shared understanding: responses cannot be cached, retries are unsafe to automate and error rates read as zero. Return an accurate [status code](./Status%20code.md).

REST can be awkward when clients need differently shaped data, leading to several round trips or oversized responses. [GraphQL](./GraphQL.md) addresses that case, and [gRPC](./gRPC.md) is common for internal service calls. An [OpenAPI](./OpenAPI.md) document is the usual way to describe a REST API.

_Usage:_

"Should cancelling an order be a DELETE?"

"No, the order still exists afterwards. In REST terms, either PATCH its status or POST to a `/orders/42/cancellation` resource."

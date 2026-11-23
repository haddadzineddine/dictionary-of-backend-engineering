---
description: A standard machine-readable format for describing HTTP APIs, used to generate docs, clients and request validation.
---

OpenAPI is a specification for describing an [HTTP](./HTTP.md) API in a YAML or JSON file. The document lists paths, [HTTP methods](./HTTP%20method.md), parameters, request and response bodies, [status codes](./Status%20code.md) and security schemes. It was formerly called Swagger, and that name survives in tools such as Swagger UI.

```yaml
paths:
  /orders/{id}:
    get:
      parameters:
        - { name: id, in: path, required: true, schema: { type: string } }
      responses:
        "200":
          description: The order
```

Because the file is machine-readable, one description can drive several tools: interactive documentation, generated client SDKs, mock servers, request validation and automated checks. It is most often used with [REST](./REST.md) style APIs.

Teams usually choose one of two workflows:

| Workflow   | Source of truth | Benefit                                | Risk                                 |
| ---------- | --------------- | -------------------------------------- | ------------------------------------ |
| Spec-first | The OpenAPI file | API is reviewed before code is written | Code can diverge without enforcement |
| Code-first | Annotated code  | Spec is generated, so less to maintain | Design follows implementation details |

The main failure is drift, where the document stops matching the running service. A consumer generates a client from the spec, ships it, then gets a validation error because a field documented as a number is actually returned as a string. Hand-maintained documents drift unless something checks them.

Reduce drift by generating the spec from code, or by validating real requests and responses against it in tests. A [contract test](./Contract%20test.md) can use the document as the agreed interface. Comparing the spec between commits in CI, the automated build pipeline, can also flag a [breaking change](./Breaking%20change.md) before it merges.

An OpenAPI file describes the shape of an API, and it says little about behaviour. It will not capture ordering guarantees, rate-limit strategy or which fields depend on each other unless you write that in its descriptions.

_Usage:_

"The generated client crashes because `amount` comes back as a string."

"So the OpenAPI spec has drifted from the service. Add response validation to the API tests so this fails in CI."

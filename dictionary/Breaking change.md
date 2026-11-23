---
description: A change to an interface that makes correct existing clients fail or misbehave without any change on their side.
---

A breaking change is a modification to an interface that causes a previously correct client to fail or behave incorrectly. The symptom usually appears as a deploy that passed all of the service's own tests, followed by errors in a different team's application.

Some breaks are obvious and others are not:

| Change                                     | Breaking? | Reason                                        |
| ------------------------------------------ | --------- | --------------------------------------------- |
| Add an optional request field              | No        | Existing requests remain valid                |
| Add a response field                       | Usually no | Strict parsers may reject unknown fields     |
| Remove or rename a field                   | Yes       | Clients still read the old name               |
| Change a type, such as number to string    | Yes       | Parsing or arithmetic fails                   |
| Make an optional request field required    | Yes       | Existing requests are rejected                |
| Tighten validation                         | Yes       | Previously accepted input now fails           |
| Add an enum value                          | Often     | Exhaustive client code hits an unknown case   |
| Change a default, sort order or [status code](./Status%20code.md) | Often | Clients may rely on it      |

The last two rows catch people out. Clients depend on observed behaviour as well as on documentation, so an undocumented sort order can become something a consumer relies on. Whether a change breaks anyone depends on actual usage.

The safe way to make an incompatible change is expand and contract:

| Step | Action                                           |
| ---- | ------------------------------------------------ |
| 1    | Add the new field or endpoint beside the old one |
| 2    | Serve both and migrate consumers                 |
| 3    | Measure until the old one has no traffic         |
| 4    | Remove the old one                               |

When this is impractical, publish the change under [API versioning](./API%20versioning.md). Catch breaks before release by diffing an [OpenAPI](./OpenAPI.md) or [Protocol Buffers](./Protocol%20Buffers.md) schema in CI, the automated build pipeline, and by running a [contract test](./Contract%20test.md) for each known consumer. Note that during a rolling deploy, old and new versions run at once, so even internal changes must be compatible across that window.

_Usage:_

"We only renamed `userId` to `user_id`. Why is mobile crashing?"

"That's a breaking change. Old app versions still read `userId`, so return both until they've gone."

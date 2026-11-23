---
description: A simple object that carries data across a boundary, keeping the API's shape separate from internal models.
---

A DTO (data transfer object) is a simple object that carries data across a boundary, such as between an API and application code. It holds fields and has no business behaviour. Its purpose is to make the shape of data at the boundary explicit and independent from internal models.

Without a DTO, handlers often serialize the database model directly. This causes two well-known security failures:

| Failure            | What happens                                                            | DTO fix                       |
| ------------------ | ----------------------------------------------------------------------- | ----------------------------- |
| Data exposure      | A new `password_hash` or `internal_notes` column appears in API responses | Response DTO lists allowed fields |
| Mass assignment    | A client adds `"is_admin": true` to a profile update and it gets saved  | Request DTO omits that field  |

Both happen quietly. Nobody edits the endpoint: someone adds a column, and the API starts returning it. An explicit allowlist of fields makes exposure a deliberate decision and supports field-level [authorization](./Authorization.md).

```typescript
type CreateUserRequest = { email: string; name: string };
type UserResponse = { id: string; email: string; name: string };
```

Separate DTOs for requests and responses are normally clearer, because their fields differ. A client should not send an `id` or `createdAt`, while a response contains them.

DTOs also protect the public contract. If a response is tied to an [ORM](./ORM.md) entity, renaming a column becomes a [breaking change](./Breaking%20change.md) for API clients. With a DTO, you change the mapping and leave the API alone.

Validate request DTOs at the edge of the system. Check types, required fields and sizes before the data reaches business code, using a schema library where possible. Remember that a type annotation alone usually has no runtime effect, as with TypeScript types, so incoming JSON still needs validation.

The cost is mapping code and some repetition. For an internal tool with few fields, that can outweigh the benefit, but a public API usually justifies it.

_Usage:_

"Why is `password_hash` in the user API response?"

"The controller returns the ORM entity directly. Add a response DTO that lists only the public fields."

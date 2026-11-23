---
description: The verb on an HTTP request, such as GET or POST, stating what the client wants done and whether a retry is safe.
---

An [HTTP](./HTTP.md) method is the verb at the start of a request. It tells the server what kind of operation the client intends, and it tells every system in between, such as caches and proxies, how that request may be treated.

Two properties matter. A safe method does not ask the server to change anything. An idempotent method can be sent several times with the same intended effect as sending it once.

| Method  | Typical use             | Safe | Idempotent |
| ------- | ----------------------- | ---- | ---------- |
| GET     | Read a resource         | Yes  | Yes        |
| HEAD    | Read headers only       | Yes  | Yes        |
| OPTIONS | Ask what is supported   | Yes  | Yes        |
| PUT     | Replace a resource      | No   | Yes        |
| DELETE  | Remove a resource       | No   | Yes        |
| POST    | Create or trigger       | No   | No         |
| PATCH   | Partially update        | No   | Not by default |

These properties are promises your handler has to keep, because the protocol does not enforce them. A `GET /unsubscribe?id=42` link that changes data looks harmless until a link previewer, crawler or email scanner fetches it and unsubscribes the user with no click. State changes belong behind a non-safe method, which also supports [CSRF](./CSRF.md) defences.

Idempotency decides what can be retried automatically. Clients and proxies may resend a GET or PUT after a dropped connection. Retrying a POST can create a second order or a second charge, because the client cannot tell whether the first attempt succeeded before the connection died. An [idempotency key](./Idempotency%20key.md) makes such a POST safe to retry.

DELETE stays idempotent even if the second call returns 404, since idempotency describes the state of the server and not the response.

_Usage:_

"Half our trial users got unsubscribed and none of them clicked anything."

"The unsubscribe link is a GET, so the mail scanner triggered it. Make the link open a page and do the change with a POST."

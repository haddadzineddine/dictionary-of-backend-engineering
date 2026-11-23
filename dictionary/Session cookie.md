---
description: A browser cookie holding a random ID that points to login state stored on the server.
---

A session cookie is a browser cookie containing a random identifier that maps to login state on the server. After [authentication](./Authentication.md), the server creates a session record and sends its ID with a `Set-Cookie` header. The browser attaches the cookie to later requests, and the server looks up the session to identify the user.

Because the state stays on the server, revocation is immediate: delete the record and the cookie stops working. A self-contained [JWT](./JWT.md) does not offer this by default. The cost is a lookup on each request and a store shared by all server instances. If sessions live in one process's memory, users appear to be logged out at random once a second instance is added or the app restarts.

Cookie attributes control exposure:

| Attribute         | Effect                                      | Recommended                          |
| ----------------- | ------------------------------------------- | ------------------------------------ |
| `HttpOnly`        | JavaScript cannot read the cookie           | Always                               |
| `Secure`          | Sent only over HTTPS                        | Always in production                 |
| `SameSite`        | Limits sending on cross-site requests       | `Lax` or `Strict`                    |
| `Path` / `Domain` | Limits where the cookie is sent             | Omit `Domain` to stay host-only      |
| `Max-Age`         | Sets a lifetime                             | As short as the product allows       |

`HttpOnly` reduces the damage of cross-site scripting (attacker JavaScript running on your page), since the script cannot steal the ID, though it can still act within the page. `SameSite` helps against [CSRF](./CSRF.md), but state-changing routes should still have a dedicated defence.

Generate a long ID with a cryptographically secure random generator. Issue a new ID at login and on privilege changes, which prevents session fixation, where an attacker plants a known ID before the victim logs in. On logout, delete the record on the server as well as clearing the cookie. Apply both an idle timeout and an absolute expiry.

_Usage:_

"Since we added a second instance, people keep getting logged out."

"Session cookies point to in-process memory, so the other instance can't find them. Move sessions to a shared store."

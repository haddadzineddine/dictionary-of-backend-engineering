---
description: A long random secret that identifies a calling application or account, used for server-to-server access to an API.
---

An API key is a long random secret that a program sends to identify itself to an API. It is commonly used for server-to-server integrations and scripts, where no person is present to complete a login. A key provides simple [authentication](./Authentication.md) of the calling project or account, and it usually stays valid until someone revokes it. That long lifetime makes leaks serious.

Key handling by stage:

| Stage    | What to do                                                                     |
| -------- | ------------------------------------------------------------------------------ |
| Create   | Generate at least 128 random bits with a cryptographically secure generator    |
| Display  | Show the full key once, and keep only a short prefix for identification        |
| Store    | Save a hash of the key, never the plaintext                                    |
| Send     | Use a request header, over [TLS](./TLS.md)                     |
| Limit    | Apply scopes, per-key rate limits and optional IP restrictions                 |
| Rotate   | Allow two active keys so clients can switch without downtime                   |
| Revoke   | Disable immediately and record last-used time                                  |

Never put keys in URLs. Query strings end up in access logs, browser history, proxies and `Referer` headers. Never embed one in frontend or mobile code either, since anyone can extract it from the bundle.

Because generated keys have high entropy, a fast hash such as SHA-256 is acceptable for storage, unlike human-chosen passwords, which need slow [password hashing](./Password%20hashing.md).

A recognisable prefix, such as `acme_live_`, helps people tell test keys from live keys and lets secret-scanning tools detect leaks. If a key is committed to a repository, revoke it.

An API key identifies an application and not a person. For actions on behalf of individual users, use [OAuth 2.0](./OAuth%202.0.md). Apply [authorization](./Authorization.md) to every key, so a read-only integration cannot write.

_Usage:_

"Someone pushed our API key to a public repo, but I've already removed the commit."

"That doesn't help. Assume it's compromised, so revoke it, issue a new one and check its usage logs."

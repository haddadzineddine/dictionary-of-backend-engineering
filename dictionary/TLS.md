---
description: The protocol that encrypts traffic and proves a server's identity using certificates. It is the S in HTTPS.
---

TLS (Transport Layer Security) is the protocol that encrypts a connection and lets the client verify which server it is talking to. [HTTP](./HTTP.md) sent over TLS is HTTPS. It provides confidentiality, so observers cannot read the traffic; integrity, so changes are detected; and server [authentication](./Authentication.md) through a certificate.

A connection begins with a handshake:

| Step | Who    | What happens                                                         |
| ---- | ------ | -------------------------------------------------------------------- |
| 1    | Client | Sends supported versions, cipher suites and key-agreement data       |
| 2    | Server | Chooses parameters and sends its certificate chain                   |
| 3    | Client | Checks the chain, expiry date and that the hostname matches          |
| 4    | Both   | Derive shared session keys                                           |
| 5    | Both   | Encrypt application data with those keys                             |

TLS 1.3 completes this in one round trip, and TLS 1.2 takes two. Use TLS 1.2 or later, since 1.0 and 1.1 are deprecated. [Keep-alive](./Keep-alive.md) avoids repeating the handshake on every request.

Most TLS incidents are certificate problems. An expired certificate takes a service down for every client at once, so automate renewal and alert on the expiry date. A server that omits its intermediate certificate can work in a browser, which may already have it cached, and fail with `unable to verify` from a server-side client. A hostname mismatch fails when a service is called by an address the certificate does not list.

Do not fix these errors by disabling verification, such as `verify=False` or `NODE_TLS_REJECT_UNAUTHORIZED=0`. The traffic stays encrypted, but the client will accept an attacker's certificate. Install the correct CA certificate (the issuer's root of trust) instead.

TLS protects data only in transit and only between its two endpoints. If a proxy terminates TLS, the hop behind it is plaintext unless it is encrypted separately.

_Usage:_

"The Python client throws a certificate error, but the site is fine in Chrome."

"The server's probably missing its intermediate cert. Fix the chain and leave TLS verification on."

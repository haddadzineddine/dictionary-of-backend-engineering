---
description: A signed, self-contained token carrying claims a server can verify without a lookup. Readable by anyone unless encrypted.
---

A JWT (JSON Web Token) is a compact token containing JSON claims and a signature. It has three base64url-encoded parts separated by dots: `header.payload.signature`. A server holding the right key can verify the signature and trust the claims without a database lookup, which makes JWTs common for API [authentication](./Authentication.md) and between services.

```json
{ "iss": "https://auth.example.com", "aud": "orders-api", "sub": "user_42", "exp": 1798765432 }
```

A standard signed JWT is encoded and not encrypted. Anyone holding the token can read its payload, so never put secrets or sensitive personal data in it. The signature only proves that the contents have not changed since a trusted key signed them.

Verification must check all of the following, preferably using a maintained library:

| Check           | Failure it prevents                                                       |
| --------------- | ------------------------------------------------------------------------- |
| Signature       | Forged or edited tokens                                                   |
| Allowed `alg`   | `none` tokens and confusion between public-key and shared-secret algorithms |
| `exp` and `nbf` | Expired or not-yet-valid tokens                                           |
| `iss`           | Tokens from another issuer                                                |
| `aud`           | A token issued for one service being replayed against another             |

Configure the accepted algorithm on the server and never take it from the token's header. Decoding is not verifying: many libraries have a separate `decode` function that skips the signature entirely.

The main operational drawback is revocation. A signed token stays valid until `exp`, so a fired employee or a stolen token keeps working after you remove the account. Use short-lived access tokens, often minutes, with a revocable refresh token. A denylist of token IDs also works, but it brings back a per-request lookup.

For ordinary browser logins, a [session cookie](./Session%20cookie.md) is usually simpler and can be revoked immediately. If a browser must hold a JWT, avoid `localStorage`, which any injected script can read.

_Usage:_

"We disabled her account an hour ago. Why can she still call the API?"

"Her JWT is valid until it expires. Shorten the access token lifetime and check revocation at refresh."

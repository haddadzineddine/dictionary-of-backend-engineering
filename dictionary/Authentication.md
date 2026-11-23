---
description: Verifying who is making a request, using something like a password, passkey, session or token.
---

Authentication is the process of verifying who is making a request. It is separate from [authorization](./Authorization.md), which decides what that identity may do. A failed authentication normally returns [status code](./Status%20code.md) 401, and a known user who lacks permission gets 403.

Authentication usually happens twice. First, a user proves their identity at login. The server then issues a credential that proves it on later requests, because [HTTP](./HTTP.md) keeps no state between them.

| Credential                              | Typical caller           | Revocation                          |
| --------------------------------------- | ------------------------ | ----------------------------------- |
| [Session cookie](./Session%20cookie.md) | Browser users            | Immediate, by deleting the session  |
| [JWT](./JWT.md) access token            | APIs and services        | Usually waits for expiry            |
| [API key](./API%20key.md)               | Scripts and integrations | Immediate, by deleting the key      |
| Client certificate                      | Internal services        | Through certificate revocation or short lifetimes |

Login endpoints need specific defences. Store passwords only with [password hashing](./Password%20hashing.md). Rate-limit attempts per account and per source, because attackers replay leaked credentials from other sites. Offer multi-factor authentication. Passkeys and hardware keys resist phishing, authenticator codes are weaker, and SMS is weaker still.

Avoid revealing which accounts exist. A form that says "no account with that email" lets an attacker test a list of addresses. Return the same message for a wrong email and a wrong password, and apply the same approach to password reset and signup flows. Response time can leak the same fact, so hash a dummy value when the account does not exist.

Issue a new session identifier at login, invalidate sessions on logout and password change, and ask for the password again before sensitive actions such as changing an email address. Where possible, use a maintained library or an identity provider through [OpenID Connect](./OpenID%20Connect.md) instead of writing this yourself.

_Usage:_

"Our login page says 'unknown email' when the address isn't registered."

"That leaks which accounts exist. Authentication errors should read the same for a bad email and a bad password."

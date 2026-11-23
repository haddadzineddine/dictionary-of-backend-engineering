---
description: A framework that lets an app access a user's resources on another service with a scoped token, without seeing their password.
---

OAuth 2.0 is an [authorization](./Authorization.md) framework that lets an application act on a user's resources at another service without receiving the user's password. A scheduling app can request access to your calendar, you approve it on the calendar provider's site, and the app receives a limited token. OAuth calls the parties the resource owner (user), client (app), authorization server (issues tokens) and resource server (the API).

For apps acting on behalf of a user, use the authorization code flow with PKCE (Proof Key for Code Exchange):

| Step | Who                  | What happens                                                     |
| ---- | -------------------- | ---------------------------------------------------------------- |
| 1    | Client               | Creates a secret `code_verifier`, a `state` value and a hash of the verifier |
| 2    | Client               | Redirects the browser with scopes, `state` and the hash          |
| 3    | User                 | Logs in and approves                 |
| 4    | Authorization server | Redirects back with a short-lived, one-time code                 |
| 5    | Client               | Checks `state`, then exchanges the code plus verifier for tokens |
| 6    | Client               | Calls the API with the access token                              |

PKCE ensures that an intercepted code cannot be redeemed without the verifier. The `state` value ties the response to the browser session that started it, which guards against [CSRF](./CSRF.md).

Register redirect URIs and match them exactly, because loose matching lets an attacker receive codes. Request the narrowest scopes. Keep client secrets and refresh tokens on the server. Do not use the implicit or password grants, which current guidance rules out. For service-to-service calls with no user, use the client credentials grant.

OAuth does not by itself tell an app who logged in. An access token is meant for the API and is not proof of identity. For sign-in, use [OpenID Connect](./OpenID%20Connect.md). Access tokens may be opaque or a [JWT](./JWT.md), and the client should not depend on their format.

_Usage:_

"Can we just use the Google access token to log people in?"

"No, OAuth 2.0 access tokens aren't proof of login. Use OpenID Connect and validate the ID token."

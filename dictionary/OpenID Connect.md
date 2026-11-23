---
description: An identity layer on OAuth 2.0 that adds a verifiable ID token, enabling sign-in through an external identity provider.
---

OpenID Connect (OIDC) is an identity layer built on [OAuth 2.0](./OAuth%202.0.md). OAuth grants an app access to an API, while OIDC also tells the app who the user is. It powers "Sign in with Google" style logins and enterprise single sign-on, where one account works across many applications.

An app requests the `openid` scope during the OAuth code flow. Along with an access token, the identity provider returns an ID token, a [JWT](./JWT.md) containing claims about the login event.

| Token         | Audience        | Used for                         |
| ------------- | --------------- | -------------------------------- |
| ID token      | Your client app | Establishing who logged in       |
| Access token  | The API         | Calling that API                 |
| Refresh token | Token endpoint  | Getting new tokens without login |

Do not swap their roles. Sending an ID token to an API as a bearer credential, or treating an access token as proof of login, breaks the audience checks that keep tokens from being misused.

Before starting a session, validate the ID token with a maintained library:

| Check       | Requirement                                                     |
| ----------- | --------------------------------------------------------------- |
| Signature   | Verified against the provider's published keys                  |
| `iss`       | Exactly matches the expected issuer                             |
| `aud`       | Contains your client ID                                         |
| `exp`       | Has not passed                                                  |
| `nonce`     | Matches the value you sent, which blocks replayed tokens        |

Identify users by the pair of `iss` and `sub`, and do not use the email address as the key. Emails change, can be reassigned, and may be unverified at some providers. Automatically merging accounts on a matching email is a known route to account takeover: an attacker registers the victim's address at a provider that does not verify it, then signs in as them.

After validation, the app normally creates its own [session cookie](./Session%20cookie.md). Local [authorization](./Authorization.md) remains your responsibility, since a valid login says nothing about what the user may do in your system.

_Usage:_

"Two users share a profile after signing in with different providers."

"We're matching on email. With OpenID Connect, key accounts on issuer plus `sub`."

---
description: An attack where a malicious site makes a victim's browser send an authenticated request, using cookies it attaches automatically.
---

CSRF (cross-site request forgery) is an attack where a malicious website causes a victim's browser to send a request to another site where they are logged in. Browsers can attach cookies automatically, so the target receives a request carrying a valid [session cookie](./Session%20cookie.md) and treats it as genuine. The attacker cannot read the response, but the action still happens.

```html
<form action="https://bank.example/transfer" method="POST">
  <input type="hidden" name="to" value="attacker" />
  <input type="hidden" name="amount" value="500" />
</form>
<script>document.forms[0].submit()</script>
```

Layer these defences for cookie-authenticated, state-changing routes:

| Defence                       | How it works                                                  | Limit                                    |
| ----------------------------- | ------------------------------------------------------------- | ---------------------------------------- |
| Anti-CSRF token               | Server checks an unpredictable per-session token on each write | Must cover every state-changing route   |
| `SameSite=Lax` or `Strict`    | Browser withholds the cookie on cross-site POSTs              | Sibling subdomains count as same-site    |
| `Origin` / `Sec-Fetch-Site`   | Server rejects requests from untrusted origins                | Decide how to treat a missing header     |
| Correct [HTTP methods](./HTTP%20method.md) | GET never changes state                          | `Lax` still sends cookies on top-level GET |

Set `SameSite` explicitly and do not rely on browser defaults, which vary. Use your framework's built-in protection where it exists, since a common failure is a developer disabling it after hitting a "CSRF token missing" error on a form.

CORS (the browser rules for cross-origin reads) is not a CSRF defence. It controls whether a page can read a response, and a simple cross-site form POST is still sent.

APIs that authenticate only with an `Authorization` header are generally not exposed to CSRF, because browsers do not attach that header automatically. Cross-site scripting defeats these protections, since attacker code running on your page can read tokens and make same-origin requests. Protect login forms as well, because login CSRF can sign a victim into the attacker's account.

_Usage:_

"This POST fails with 'CSRF token missing'. Can I exempt the route?"

"No, it changes account settings. Send the token from the form instead of turning off CSRF protection."

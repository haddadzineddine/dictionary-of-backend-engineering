---
description: An attack that tricks your server into requesting a URL of the attacker's choice, often reaching internal systems.
---

SSRF (server-side request forgery) is a vulnerability where an attacker makes your server send a request to a destination they choose. It appears in features that fetch a user-supplied URL: link previews, image imports, PDF generators and [webhook](./Webhook.md) destinations. The request originates inside your network, so it can reach things the attacker cannot reach directly, including admin panels, internal APIs and cloud metadata endpoints.

Many cloud platforms expose instance information at the link-local address `169.254.169.254`, sometimes including temporary credentials. If an "import image from URL" feature fetches that address and returns the body, the attacker may obtain those credentials.

Blocklisting strings such as `localhost` is not sufficient. Attackers can use alternate IP encodings, IPv6 forms, a DNS name resolving to an internal address, or a public URL that redirects inward. Apply layered controls:

| Control                  | What to do                                                                |
| ------------------------ | ------------------------------------------------------------------------- |
| Allowlist                | Permit only known hosts wherever the feature allows it                    |
| Scheme and port          | Allow only `https`, or `http` if required, on expected ports              |
| Resolved IP check        | Resolve the name and reject private, loopback, link-local and mapped addresses |
| Pin the connection       | Connect to the validated IP, which prevents DNS rebinding                 |
| Redirects                | Disable them, or revalidate every hop                                     |
| Network egress           | Run fetchers in an isolated network with outbound rules                   |
| Metadata hardening       | Require session-based metadata access, such as AWS IMDSv2                 |
| Response handling        | Enforce size and time limits, and do not return raw bodies or errors      |

DNS rebinding is why checking a URL once is not enough: the name resolves to a public address during validation and an internal one at fetch time.

Return one generic failure, since differing errors let attackers map internal ports. Internal services should still require [authentication](./Authentication.md) instead of trusting a request because it came from inside.

_Usage:_

"The link preview fetched `169.254.169.254` during the pen test."

"That's SSRF. Validate the resolved IP, pin the connection and block redirects."

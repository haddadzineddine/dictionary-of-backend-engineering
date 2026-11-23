---
description: Deciding whether an already identified caller may perform a specific action on a specific resource.
---

Authorization is the decision about whether a known caller may perform a specific action on a specific resource. It runs after [authentication](./Authentication.md), which establishes who the caller is. When the check fails, the server normally returns [status code](./Status%20code.md) 403, or 404 if the existence of the resource should stay hidden.

The most common failure is a missing check on an individual object. A logged-in user opens `/invoices/1001`, changes the number to `1002` and sees another customer's invoice. Nothing verified that this invoice belongs to this user. This flaw is known as an insecure direct object reference (IDOR), or broken object-level authorization. Unguessable IDs make it harder to exploit, but they do not fix it.

Authorization happens at several levels:

| Level    | Question                                   | Example                              |
| -------- | ------------------------------------------ | ------------------------------------ |
| Function | May this caller use this operation?        | Only admins can issue refunds        |
| Object   | May they act on this particular record?    | Users read only their own invoices   |
| Field    | May they read or set this property?        | Users cannot set their own `role`    |

Enforce every check on the server, because hiding a button changes nothing for someone calling the API directly. Deny by default, so a forgotten rule fails closed. Check on every request, including list, export and bulk endpoints, and on any [background job](./Background%20job.md) acting for a user. Scope queries to the tenant or owner, such as `WHERE id = ? AND account_id = ?`.

Keep rules in one shared place and not scattered across handlers. [RBAC](./RBAC.md) grants permissions through roles, while [ABAC](./ABAC.md) evaluates attributes of the user, resource and context. Write tests asserting that one user cannot reach another user's data, because normal feature tests only exercise the permitted path.

_Usage:_

"QA changed the order ID in the URL and got someone else's receipt."

"That's a missing authorization check. Load the order scoped to the current account and return 404 if it isn't theirs."

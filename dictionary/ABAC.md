---
description: Attribute-based access control. Access is decided by policies evaluating properties of the user, resource, action and context.
---

ABAC (attribute-based access control) is an [authorization](./Authorization.md) model where a policy decides access by evaluating attributes. Those attributes describe four things: the subject (user), the resource, the action and the environment. A rule might read: "A manager may approve an expense if it belongs to their department, is under their limit and was not submitted by them."

```text
allow approve on expense if
  user.department == expense.department
  and expense.amount <= user.approval_limit
  and expense.submitter_id != user.id
```

Expressing this with [RBAC](./RBAC.md) alone would need a role per department and limit, which is the role explosion that usually prompts teams to adopt ABAC.

| Aspect            | RBAC                               | ABAC                                   |
| ----------------- | ---------------------------------- | -------------------------------------- |
| Decision based on | Assigned roles                     | Attributes and policy rules            |
| Granularity       | Coarse                             | Fine, down to individual records       |
| Answering "who can access this?" | Read role assignments | Must evaluate policies against users   |
| Implementation effort | Lower                          | Higher                                 |
| Good fit          | Stable job functions               | Ownership, region, time or data class  |

ABAC is only as trustworthy as its attributes. If a user can edit their own `department` through a profile endpoint, they can grant themselves access, so attributes used in decisions must come from a trusted source. Treat a failed attribute lookup or an unmatched rule as a denial.

List endpoints are the usual implementation difficulty. Loading every row and filtering it in application code is slow and risks leaking data through counts or [pagination](./Pagination.md). The policy needs to be translated into a query filter, which some policy engines support directly.

Policy engines such as Open Policy Agent and Cedar keep rules outside application code, so they can be reviewed and tested. Because a small policy edit can widen access silently, add tests for allowed and denied cases and log each decision.

_Usage:_

"Managers should only approve expenses from their own team, and never their own."

"That doesn't fit a role. Write it as an ABAC policy on department and submitter."

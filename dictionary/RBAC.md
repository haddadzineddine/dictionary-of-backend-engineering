---
description: Role-based access control. Permissions are granted to roles, and users receive permissions by being assigned roles.
---

RBAC (role-based access control) is an [authorization](./Authorization.md) model where permissions are attached to roles and users are assigned roles. Instead of granting "can refund orders" to forty people individually, you grant it to `support_lead` and assign that role.

| Role          | `orders:read` | `orders:refund` | `users:manage` |
| ------------- | ------------- | --------------- | -------------- |
| `viewer`      | Yes           | No              | No             |
| `support`     | Yes           | No              | No             |
| `support_lead`| Yes           | Yes             | No             |
| `admin`       | Yes           | Yes             | Yes            |

Code should check permissions and not role names. A check such as `if user.role == "admin"` spreads through the codebase, and adding a new role later means finding every one of those conditions. `require("orders:refund")` keeps the mapping in one place, so a role can change without code changes.

In a product with multiple customers, a role assignment must be scoped to a tenant. A user may be `admin` in one organisation and `viewer` in another. If the role is stored globally on the user, an admin of a small test account can gain admin rights over everyone's data.

RBAC answers "may this kind of user do this kind of thing?" It does not on its own answer "is this their invoice?" You still need an object-level ownership check. The usual symptom of stretching it too far is role explosion: `manager_eu`, `manager_eu_readonly`, `manager_us_contractor`, until nobody can say what each role grants. That is a sign that the rule depends on attributes, which is the territory of [ABAC](./ABAC.md). Many systems use roles for broad access and attribute checks for the exceptions.

Apply least privilege: give the smallest role that does the job, review assignments regularly and remove them when people change teams. Log every role change, since it is a security-relevant event.

_Usage:_

"We've got 140 roles and nobody knows what half of them do."

"RBAC is being used to encode region and team. Keep a few roles and move those conditions into attribute rules."

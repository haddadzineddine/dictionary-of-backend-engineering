---
description: An attack where input is interpreted as part of a SQL statement. Prevented by passing values as bound parameters.
---

SQL injection is a vulnerability where untrusted input becomes part of the text of a database query, so the database runs it as code. It can expose or alter data and bypass login, and on some configurations it can reach the underlying server. An early, harmless sign is a search or signup form that returns a 500 when someone named O'Brien uses it: the apostrophe has ended the string in the query.

```python
# Vulnerable: input is concatenated into the statement
db.execute(f"SELECT * FROM users WHERE email = '{email}'")

# Safe: the value is sent separately as a bound parameter
db.execute("SELECT * FROM users WHERE email = %s", (email,))
```

With a parameterized query, the driver or database treats the value strictly as data, whatever characters it contains. This is the primary defence. Manual escaping and blocklists of words such as `DROP` are unreliable and should not be used in its place.

Parameters do not cover every part of a query:

| Query part                | Safe approach                                             |
| ------------------------- | --------------------------------------------------------- |
| Values                    | Bound parameters                                          |
| Column or table names     | Map user input to a hardcoded allowlist                   |
| Sort direction            | Allow only `ASC` or `DESC`                                |
| `IN (...)` lists          | Generate one placeholder per value, or bind an array      |
| Raw queries in an [ORM](./ORM.md) | Use the ORM's parameter binding, never interpolation |
| Dynamic SQL in stored procedures | Bind parameters there too                          |

An ORM protects its normal query methods, but its raw-query features can still be injected. Stored data can also be dangerous: in second-order injection, a value saved safely is later concatenated into another query, so treat values read from your own database as untrusted too.

Add defence in depth. Connect with a database account holding the least privileges it needs, so that an injection cannot drop tables. Return generic error messages, since raw database errors help attackers refine a payload. Input validation is useful but does not replace parameters.

_Usage:_

"Search crashes whenever the term contains an apostrophe."

"That's SQL injection waiting to happen. Replace the string formatting with a bound parameter."

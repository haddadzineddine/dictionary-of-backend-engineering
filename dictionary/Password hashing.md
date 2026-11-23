---
description: Storing passwords through a slow, salted, one-way function so a stolen database does not reveal them.
---

Password hashing is storing the output of a slow, salted, one-way function in place of the password itself. At login, the server applies the same function to the submitted password and compares results. If the database leaks, attackers still have to guess each password individually.

The function is one-way, so the output cannot be reversed. A salt, a random value unique to each password and stored beside the hash, means identical passwords produce different hashes and precomputed tables are useless. The function is deliberately expensive, which limits how many guesses per second an attacker can make.

General-purpose hashes such as MD5, SHA-1 and SHA-256 are designed to be fast, so they are the wrong tool here, even with a salt. Encryption is also wrong, since anyone who obtains the key can recover every password.

OWASP's password storage guidance lists these minimum settings, which you should recheck periodically:

| Algorithm          | Minimum configuration                   | Notes                          |
| ------------------ | --------------------------------------- | ------------------------------ |
| Argon2id           | 19 MiB memory, 2 iterations, p=1 | First choice                   |
| scrypt             | N=2^17, r=8, p=1                        | Use if Argon2id is unavailable |
| bcrypt             | Work factor 10 or more                  | Input limited to 72 bytes      |
| PBKDF2-HMAC-SHA256 | 600,000 iterations                      | For FIPS-compliance needs      |

Use a maintained library and its verify function, which handles salts and constant-time comparison. Its encoded output records the algorithm and parameters, so you can raise the cost later and rehash at the next successful login.

Slow hashing has a server-side cost. A burst of login attempts can saturate the CPU and, in a single-threaded runtime, stall the [event loop](./Event%20loop.md) for every user. Rate-limit login, run hashing off the main [thread](./Thread.md), and set a generous maximum password length so very long inputs cannot be used for denial of service.

_Usage:_

"The audit flagged our password column. It's SHA-256 with a salt, isn't that fine?"

"No, it's too fast. Move to Argon2id password hashing and rehash each user at next login."

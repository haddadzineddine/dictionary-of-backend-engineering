---
description: Returning a large result set in smaller pieces so that no single response has to load, serialize and send everything.
---

Pagination is splitting a large list into smaller responses that a client requests one at a time. Without it, an endpoint such as `GET /orders` works in development with 20 rows, then times out or runs the server out of memory once a customer has several hundred thousand. The server has to load, serialize and send everything in one response.

There are three common approaches:

| Approach                                    | Request                  | Strength                      | Weakness                               |
| ------------------------------------------- | ------------------------ | ----------------------------- | -------------------------------------- |
| Offset                                      | `?limit=50&offset=100`   | Simple, can jump to any page  | Slows with depth, shifts during writes |
| Page number                                 | `?page=3&per_page=50`    | Maps to numbered page UI      | Same as offset                         |
| [Cursor](./Cursor%20pagination.md)          | `?limit=50&after=abc123` | Stable and fast at any depth  | No jumping to an arbitrary page        |

Offset pagination has two known problems. The database generally has to walk past all skipped rows, so page 10,000 is much slower than page 1. Results also shift when data changes between requests. If a row is inserted at the top while a client is reading, the last item of page 1 reappears as the first item of page 2. If a row is deleted, an item gets skipped.

Some rules apply to any approach. Always enforce a server-side maximum page size, because a client-supplied `limit=1000000` removes the protection. Sort by a deterministic order that ends in a unique column, such as `created_at, id`. Without a tie-breaker, rows sharing a timestamp can appear on two pages or on none.

Be careful with total counts. A `total` field looks cheap, but counting a large filtered set can cost more than fetching the page. Consider omitting it, approximating it or returning only a `has_more` flag. Changing the pagination style of a published endpoint is usually a [breaking change](./Breaking%20change.md).

_Usage:_

"The export job keeps seeing the same invoice twice."

"It's using offset pagination while new invoices come in. Switch it to a cursor on `created_at` and `id`."

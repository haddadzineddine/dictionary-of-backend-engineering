---
description: Fetching the next page by remembering the last item seen, which stays fast and stable while data changes.
---

Cursor [pagination](./Pagination.md) is a method where each request says "give me the items after this one" instead of "skip N items". The server returns a page and a cursor, an opaque token that marks the position of the last item. The client sends that token back to get the next page.

```sql
SELECT id, created_at, total
FROM orders
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 51;
```

The cursor here encodes the `created_at` and `id` of the last row. Some databases need this row comparison written out. With an index on `(created_at, id)`, the database jumps to the position directly, so the thousandth page costs about the same as the first. Inserts and deletes elsewhere do not shift the position, which removes the duplicated and skipped rows seen with offsets.

| Step | Who    | What happens                                                    |
| ---- | ------ | --------------------------------------------------------------- |
| 1    | Client | Requests the first page with a limit of 50                      |
| 2    | Server | Fetches 51 rows, so the extra row shows whether more exist      |
| 3    | Server | Returns 50 rows and a cursor built from the 50th                |
| 4    | Client | Sends the cursor to fetch the next page                         |
| 5    | Server | Decodes and validates it, then continues from that position     |

The sort must end in a unique column. If you sort by `created_at` alone, rows with the same timestamp at a page boundary get lost.

Keep cursors opaque, for example base64-encoded, so clients do not depend on their contents and you can change the format later. Encoding is not protection, so treat a decoded cursor as untrusted input: validate it, use bound parameters to prevent [SQL injection](./SQL%20injection.md), and apply normal [authorization](./Authorization.md) filters on every page.

The trade-off is navigation. You cannot jump to page 37, and a cursor is tied to its sort order, so changing the sort requires starting again.

_Usage:_

"Deep pages on the activity feed take seconds to load."

"That's the offset scanning past every row. Cursor pagination on `(created_at, id)` will keep it constant."

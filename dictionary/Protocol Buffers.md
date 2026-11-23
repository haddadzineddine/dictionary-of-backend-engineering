---
description: A language-neutral schema and compact binary format for structured data, identifying each field by a number instead of a name.
---

Protocol Buffers, or Protobuf, is a schema language and binary serialization format created at Google. You define messages in a `.proto` file, and a compiler generates types for each language you use. It is the default message format for [gRPC](./gRPC.md).

```proto
message Order {
  string id = 1;
  int64 total_cents = 2;
  optional string coupon = 3;
}
```

The numbers are field numbers, and they are what goes on the wire. Field names are not sent in the binary format, which keeps messages small. It also means the number is the identity of the field: two programs agree on what a value means only because they agree on its number.

This drives the rules for changing a schema safely:

| Change                     | Safe? | Why                                                         |
| -------------------------- | ----- | ----------------------------------------------------------- |
| Add a field with a new number | Yes | Old readers skip fields they do not know                    |
| Remove a field and reserve it | Yes, if readers tolerate its absence | `reserved` stops the number being reused |
| Rename a field             | Binary yes, JSON no | The number is unchanged, but JSON uses names and generated code changes |
| Change a field's type      | Usually no | Only some types are compatible, and values may be truncated |
| Reuse an old field number  | No    | Old data and old clients misread the new field              |

Reusing a number is the failure to watch for. Nothing errors. A service still running the old schema decodes the new value as the old field, producing wrong data that can go unnoticed. When deleting a field, add `reserved 3;` so the compiler rejects reuse. Treat any unsafe change as a [breaking change](./Breaking%20change.md).

In proto3, a plain scalar field cannot distinguish "unset" from its default, such as `0` or an empty string. A discount of zero and a missing discount look identical. Mark the field `optional` when that difference matters.

_Usage:_

"Old consumers are suddenly showing refund amounts as customer IDs."

"Someone reused field number 4 in the Protobuf schema. Give it a new number and mark 4 as reserved."

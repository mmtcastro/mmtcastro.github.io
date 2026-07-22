---
layout: post
title: "The case of the missing cents: a float32 trap in the Domino REST API"
date: 2026-07-22
categories: [domino, drapi, java, spring, data-integrity, debugging]
---

One day, a report closed with different numbers than the ERP. Not by much — cents. A value stored as 3,209,185.63 arrived on the other side as 3,209,185.8. No error in any log, no warning, HTTP 200 everywhere. Just cents going missing, and only on large values.

If you have ever hunted a bug like this, you know that cents on values in the millions are the worst kind of symptom: too small to scream, too large to ignore in a financial system. This post is the full story of that hunt — from first suspicion to a very unlikely culprit — on the Domino REST API (formerly Project KEEP), the REST layer I use to modernize a 25-year-old Domino ERP. It is the saga the [MCP server post]({% post_url 2026-07-22-mcp-server-domino-rest-api-en %}) promised to tell.

## The database doesn't lie

Everyone's first hypothesis: corrupted data in the database. I opened the document in the Notes client: **3,209,185.63**, correct to the last cent. I exported the document as DXL, Domino's low-level XML format, which shows the value as it lives on disk: full precision. The legacy system had always stored numbers as 64-bit floating point, and 25 years of data were intact.

But the JSON coming out of the API said 3,209,185.8. Same document. The disk wasn't lying — the serialization was. The value was being truncated on the way between the disk and the HTTP response, both when reading through a view and when fetching the document directly.

A trap inside the trap: the debugging tool lies too. If you take the response JSON and *parse* it to inspect the value, your language's parser converts the number to a 64-bit float and pretty-prints it — masking exactly what you are trying to see. The investigation only moved forward when I started inspecting the **raw text** of the response, with a regex, before any parsing. In a numeric-precision bug, look at the bytes, not the objects.

## Seven digits

The cause was in the schema. In the Domino REST API, every field declared in the schema has a `format` — and the server serializes the value **according to the declared format**, not according to what is on disk. The monetary fields were declared as `number` with `format: "float"`.

`float` means 32-bit floating point. And 32-bit floats carry about **7 significant digits**.

Do the math: 3,209,185.63 has nine significant digits. It doesn't fit. The serializer rounds to the nearest 32-bit float — 3,209,185.8 — and that is what goes out in the JSON. Small values pass through unharmed (they fit in 7 digits); values from the high hundreds of thousands upward lose their cents. That is why the symptom was selective: only the large values, only at the edges.

And here the story takes the turn that makes it worth a post. Who declared `float` on those fields? **We did.** The schema generator I used to provision the API's scopes mapped Notes numeric fields to `float` by default. This was not a product bug — it was my own provisioning tool silently re-introducing the trap into every new schema it generated. The butler did it, and the butler was on my payroll.

## The near-disaster: read, then write back

So far, a read bug. Annoying, but the data was safe. The real danger was something else, and catching it in time was equal parts luck and paranoia.

Consider the most natural cycle in any application: fetch a document through the API, change some unrelated field — a status, a comment — and save it back. The monetary value came back **truncated** on the read. If the client writes the whole document back, it writes the truncation. The disk, which was intact, now holds 3,209,185.8 *for real*. A read bug becomes permanent corruption of financial data, silently spread across every document any process touches.

It was a close call. And this is the lesson I would underline three times: in a schema-driven API, **a read bug is never just a read bug** as long as any read-then-write-back path exists.

## The fix: a schema flip, no restart

The fix, mercifully, is anticlimactic — in the best way. The `format` is a schema attribute, and the schema is editable at runtime via POST to the administration endpoint (`setup-v1`): read the schema, flip `float` to `double` on the numeric fields, write it back. No server restart, reversible, and the server does not "re-discover" the format on its own — the flip sticks. On the version I use, the effect is immediate: the same query that returned 3,209,185.8 starts returning 3,209,185.63.

What was *not* anticlimactic was making sure the fix caught **everything**. The sweep I wrote to audit and flip the fields had three blind spots, and each one cost me an extra round:

- **Time** — schemas generated *after* the sweep were born with `float` again, because the generator still had the wrong default. Fixing the schemas without fixing the generator is bailing water with the tap open.
- **Place** — scopes provisioned outside the usual path escaped the first pass.
- **Depth** — multivalue (array) fields declare their format in `items: {format}`, one level below where the sweep was looking. Scalars fixed, arrays still truncating. That was the third axis, discovered weeks later.

And one operational warning that nearly became a booby trap: **schema backups preserve the past**. An old dump restored via POST re-introduces `float` in bulk, undoing the fix in one command. A schema backup is code — treat it like code.

## The shortcut that was a dead end

Before the flip, I considered the classic shortcut: "floats are bad for money? I'll declare it as a string and format it myself." Don't. In the Domino REST API, a numeric value written against a field declared `string` in the schema is **dropped on write** — the document saves, HTTP 200, and the field simply is not there. The shortcut trades lost cents for lost values.

The rule that came out of this, now a standing directive on the project: money is `number` with `format: "double"` in the schema and `Double` in the Java client, end to end. And a one-cent difference in a value comparison is **a bug to investigate, never a tolerance to accommodate** — it was tolerance for "tiny differences" that kept this bug invisible for far too long.

## The moral: a schema is a precision contract

Three lessons, in increasing order of generality:

1. **In a schema-driven API, the schema is not just the shape — it is the precision.** The server delivers what the schema dictates, not what the disk holds. One wrong `format` corrupts data without ever touching the database.
2. **Audit your generators, not just your data.** The bug did not live in any particular schema; it lived in the schema factory. Sweeps fix the inventory; only fixing the generator stops the leak.
3. **Round-trip test your numeric fields at the start of the project.** Write a nine-significant-digit value, read it back, compare byte-for-byte against the raw JSON. A five-line test would have paid for months of hunting.

Today this whole saga has a curious epilogue: it became an entry in the compendium that the [MCP server serves as a tool]({% post_url 2026-07-22-mcp-server-domino-rest-api-en %}) — it was precisely the story that a fresh AI session, with zero prior context, retold me on its own when I tested that tool. The lesson was paid for once; now it is infrastructure.

---

The reference version of this story — Symptom → Cause → Fix, alongside the other schema and type traps — lives in [schema-types.md](https://github.com/mmtcastro/domino-rest-api-guide/blob/main/schema-types.md) in the public [domino-rest-api-guide](https://github.com/mmtcastro/domino-rest-api-guide) repo.

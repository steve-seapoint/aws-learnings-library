---
id: dynamodb-002-49
title: Single-table design is a misnomer — co-locate only for item collections
services: [DynamoDB]
summary: The only thing that justifies putting two items in one DynamoDB table is retrieving them together (an item collection via Query, or a Scan). Unrelated, independently-accessed data (sessions, telemetry, locks) belongs in its own table — table count is not a cost or design axis, and forcing everything into one table imposes one Streams/backup/capacity config on all of it, bloats items, and widens blast radius.
---

# "Single-table design" is a misnomer — co-locate only for item collections

**Problem:** A team treats "single-table design" as the goal and packs every entity —
durable domain records plus unrelated, ephemeral data like session/presence markers,
counters, locks, or operational telemetry — into one table with overloaded
`PK`/`SK` prefixes. It feels like best practice. In reality it forces awkward keys,
bloats items, makes a routine cleanup/aggregation job Scan the whole table, and blocks
per-data-type operational choices (e.g. one data type needs a Stream or different
backup policy and now *can't* have one because the shared table doesn't).

**Root Cause:** "Single-table design" was always shorthand for *denormalize and use
schema flexibility*, not "literally one table." Per Pete Naylor (ex-DynamoDB PM),
**the only compelling cost/performance reason to store two items in the same table is
that you retrieve them together** — as an *item collection* (a `Query` on a shared
partition key) or a `Scan`. Everything else people cite is a non-reason:

- `BatchGetItem` and `TransactWriteItems` **work across tables** with no metering or
  performance penalty — so needing a transaction or batch get is *not* a reason to
  co-locate.
- **Table count is not a DynamoDB cost or design axis.** You are billed on item size,
  request volume, and indexes — never on number of tables. "The optimal number of
  tables might be one, but often it will be more."

Dogmatic co-location of independently-accessed data has concrete costs:
- **Forced-shared configuration:** one table = one Streams setting, one backup/PITR
  policy, one capacity mode, one CMK — imposed on *all* the data. A data type that
  wants a Stream (or wants to skip PITR) can't get its own.
- **Wider blast radius:** a bad write, a bad migration, or a slow restore in one data
  type now touches unrelated functionality.
- **Item bloat:** key-uniqueness prefixes add bytes to every item, and you're billed
  per byte on every read and write.
- **Harder ETL / access control:** you can't selectively export or scope table-level
  policies to one data type.

**Solution:** Model by access pattern, then let the table count fall out of it.
Co-locate two items **only** if either is true — otherwise give them separate tables:

1. They're retrieved together as an item collection (`Query` on a shared partition
   key) or by `Scan`.
2. They're the same-type records owned by one service with identical operational
   requirements (Streams/backup/capacity/encryption).

```text
# Decision rule
Do these two item types get read together in one Query (or Scan)?
  ├─ yes → same table (that's what item collections are for)
  └─ no  → separate tables, aligned to service + operational boundaries
           (a cross-table BatchGetItem / TransactWriteItems is free — not a reason to merge)
```

Worked example: ephemeral session/presence rows never join any domain record, so they
gain nothing from co-location — but co-locating them means they can't have their own
Stream (if the main table is deliberately stream-less) and a periodic cleanup job must
Scan all the durable content to find them. Their own small table fixes both.

**Key Insight:** The number of tables is not a DynamoDB design axis — access patterns
are. Put two items in the same table only because you retrieve them together as an item
collection (or Scan); independently-accessed data belongs in its own table so it can
carry its own Streams, backup, and capacity configuration and keep its blast radius
small. "You'll wind up with N tables. It's okay."

Sources: Pete Naylor — "Single table design for DynamoDB: The reality"
(blog.highbar.solutions) and "What really matters in DynamoDB data modeling?"
(gomomento.com).

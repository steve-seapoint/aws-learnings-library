---
id: dynamodb-001-53
title: Design single-table DynamoDB access patterns (and their GSIs) up front
services: [DynamoDB]
summary: Single-table DynamoDB is access-pattern-first — enumerate every read (including "list all X") before modeling, and provision the GSI + stamp its key at write time. A listing that needs a Scan, or an index retrofitted onto live data, means the pattern was not designed in.
---

# Design single-table DynamoDB access patterns (and their GSIs) up front

**Problem:** A "list all X" view (a landing page listing every tenant/vault/account) is slow and expensive — or worse, a later "optimization" swaps the Scan for a Query against a new GSI, ships, and the list comes back **empty** (a silent 200 with no items) even though every record is intact. It reads like data loss. It isn't; the retrofitted index just wasn't populated for pre-existing items.

**Root Cause:** The table wasn't designed for that access pattern. In single-table design, `PK=THING#{id}` puts each entity in its **own partition**. A DynamoDB Query needs one partition-key value, so "enumerate all things across partitions" is **impossible on the base table** — it degrades to a full-table **Scan**. That Scan is the tell that a required read (collection listing) was never modeled. The usual "fix" then bolts on a **sparse / overloaded GSI** (an inverted index: a shared synthetic key like `GSI1PK=REGISTRY` on each config item, making the collection one cheap Query) — but retrofitting it onto live data needs a backfill, and shipping the Query before that backfill causes the silent-empty failure. All of it is avoidable by designing the pattern in.

**Solution:** Design access patterns first, then the keys and indexes to serve them.
1. **Enumerate every access pattern before modeling** — every read the app and its agents will make, including collection listings, filters, and sorts. This is the core single-table discipline (Rick Houlihan): you cannot key a table correctly without the full read list.
2. **Give each listing a home partition from day one.** For "list all X," provision a sparse/overloaded GSI with a shared partition key and **stamp that key at write time**, in the same create/update path that writes the item:
```python
# At entity-create time — the index key is written WITH the item, never later.
table.put_item(Item={
    "PK": f"THING#{thing_id}", "SK": "CONFIG",
    "GSI1PK": "REGISTRY", "GSI1SK": f"THING#{thing_id}",  # sparse listing index
})
# list_all_things() is then a Query, never a Scan:
#   query(IndexName="GSI1", KeyConditionExpression=Key("GSI1PK").eq("REGISTRY"))
```
3. **If you must add an index to an existing table, backfill BEFORE the query code ships** — stamp the key onto all existing items first (idempotent, additive), so the new Query never misses un-migrated rows. This is remediation, not the goal.

**Key Insight:** Know all your access patterns before you design a single-table DynamoDB schema. A collection read that requires a Scan — or a core listing index you have to retrofit and backfill — is a design smell: that pattern should have been keyed in from the start, with its GSI key written at write time.

---
id: dynamodb-001-53
title: Run the DynamoDB backfill BEFORE deploying code that changes a read pattern
services: [DynamoDB, Lambda]
summary: Switching a single-table listing from Scan to a sparse-GSI Query needs its backfill run first — new query code over un-migrated items returns a silent empty result (HTTP 200, empty list), which looks exactly like data loss but isn't.
---

# Run the DynamoDB backfill BEFORE deploying code that changes a read pattern

**Problem:** After a routine deploy, a "list all X" view suddenly showed nothing — e.g. a UI reading "No items yet" — even though every record was still in the table. It looks like catastrophic data loss. It is not: the underlying items are intact; the *index the new code queries* is empty.

**Root Cause:** The deploy changed a read pattern. In single-table DynamoDB, `PK=THING#{id}` puts each entity in its **own partition**, so "list all things" cannot be a Query on the base table (Query needs one partition key value) — the old code did a **Scan** with a filter (e.g. `SK=CONFIG`). The new code switched to the correct pattern: a **sparse / overloaded GSI** ("inverted index") — a shared synthetic partition key (`GSI1PK=REGISTRY`) stamped on each config item, so listing becomes one cheap Query. But that marker is only present on items that were **backfilled**. The migration that stamps `GSI1PK` onto pre-existing items was never run in the target environment, so the GSI partition was empty. A Query over an empty index returns a **successful 200 with an empty list — no error, no log** — the most deceptive failure mode. Rolling back the Lambda alias did **not** help: it is a data-shape mismatch, not a code regression.

**Solution:** Run the idempotent backfill to stamp the GSI key onto existing items, then verify the GSI Query returns them.
```python
# One-time, purely additive: stamp the sparse-GSI marker onto pre-existing items.
# Idempotent — only touches items MISSING the marker; never mutates other attrs.
from boto3.dynamodb.conditions import Attr
kwargs = {"FilterExpression": Attr("SK").eq("CONFIG")
          & Attr("PK").begins_with("THING#") & Attr("GSI1PK").not_exists()}
while True:
    resp = table.scan(**kwargs)
    for it in resp["Items"]:
        table.update_item(
            Key={"PK": it["PK"], "SK": "CONFIG"},
            UpdateExpression="SET GSI1PK = :pk, GSI1SK = :sk",
            ConditionExpression=Attr("GSI1PK").not_exists(),   # idempotent
            ExpressionAttributeValues={":pk": "REGISTRY", ":sk": it["PK"]})
    if "LastEvaluatedKey" not in resp: break
    kwargs["ExclusiveStartKey"] = resp["LastEvaluatedKey"]
```
Verify: `aws dynamodb query --index-name GSI1 --key-condition-expression "GSI1PK = :r" ...` returns the full set.

**Prevention:**
- **Migrations run BEFORE the code that depends on them.** Stamp-first means there is never a window where the new Query misses un-backfilled items. (Backfill scripts often say this in their own docstring — read it.)
- **Pre-deploy checklist:** is the target env behind `main`, and does the delta include a schema/index migration or backfill? A "small fix" deploy can silently drag in someone else's unmigrated read-pattern change.
- **A failed read that returns empty is worse than one that errors** — the outage is invisible until a human looks. Add a smoke check after any index/GSI change: does the collection Query return the expected count?

**Key Insight:** New query code over un-migrated data fails *silently and empty*, not loudly. For single-table DynamoDB, the correct "list a collection" pattern is a sparse GSI (not a separate table) — but shipping the Query before its backfill turns a good design into a phantom data-loss incident.

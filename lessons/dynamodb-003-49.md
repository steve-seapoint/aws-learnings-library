---
id: dynamodb-003-49
title: Don't clean up ephemeral rows with a recurring Scan — TTL deletes are free, use Streams
services: [DynamoDB]
summary: A scheduled Scan+FilterExpression to delete or aggregate a small class of rows is billed on bytes scanned across the WHOLE table (the filter runs after billing), so cost scales with total table size × frequency. TTL deletes cost zero write units; react to TTL-delete Stream events instead of polling; use S3 export for analytics, not Scan.
---

# Don't clean up ephemeral rows with a recurring Scan — TTL deletes are free, use Streams

**Problem:** A scheduled job (a cron Lambda) deletes or aggregates a small, high-churn
class of rows — session/presence markers, expired locks, staged events, counters — by
`Scan`ning the table with a `FilterExpression` that keeps just those rows. It looks
cheap and correct, and every *application* Query is properly key-targeted, so code
review passes. But the table's read cost climbs steadily as it grows, with no visible
cause, and CloudWatch shows `Scan` is the dominant consumer of read capacity.

**Root Cause:** A `Scan` **reads and bills every byte in the table** (per-4KB read
units), regardless of how few items match — and a `FilterExpression` is applied *after*
the read, so it reduces the returned result set but **not** the cost. This is exactly
the server-side compute/filtering DynamoDB is worst at. Run it on a schedule (every
5 min = 288×/day) and the bill scales with **total table size × frequency**, while the
working set it actually cares about stays tiny. Meanwhile you're paying twice for
deletion: a manual `DeleteItem` costs a write unit sized to the item, on top of the
Scan that found it.

**Solution:** Stop polling. Let DynamoDB's free primitives do the work.

1. **Reap on expiry for free.** DynamoDB **TTL deletes cost zero write units** — set a
   numeric epoch-seconds `ttl` attribute and let DynamoDB remove expired rows. Never
   Scan-and-`DeleteItem` ephemeral data.
2. **React to events, don't poll.** If the sweep also emits a metric or triggers an
   action ("row expired without being consumed"), consume the **TTL-delete DynamoDB
   Stream** event instead of scanning. Cost becomes O(events), not O(table).
   ```text
   TTL expiry → Stream record (REMOVE, userIdentity=dynamodb.amazonaws.com)
              → Lambda → emit metric / enqueue follow-up
   # No scheduled Scan. No per-item DeleteItem charge.
   ```
3. **Analytics = export, not Scan.** For periodic aggregation/reporting, use DynamoDB's
   **S3 export (~$0.10/GB)** and query in Athena — do not Scan the live table.
4. **If you must keep a periodic sweep, isolate the data first.** Move the ephemeral
   rows to their own small table so the Scan traverses only them, not your durable
   content (and that table can carry the Stream the main table may deliberately lack).

**Key Insight:** `Scan` bills on bytes *scanned*, not *returned*, and `FilterExpression`
filters after you've paid — so a recurring Scan-with-filter to clean up ephemeral rows
is billed on your entire table forever. TTL deletes are free and Streams/exports turn a
polling cost of O(table × frequency) into O(events).

Source: "Mo's Lunch & Learn: optimizing DynamoDB" (Khawaja Shams & Pete Naylor) —
TTL deletes are free; use export for analytics instead of scans.

---
id: dynamodb-004-49
title: Code review can't tell you a DynamoDB design is healthy — measure per-operation
services: [DynamoDB]
summary: A schema where every application query is perfectly key-targeted in the source can still be dominated at runtime by one scheduled Scan. Static review measures the design, not executed traffic. Before declaring a table healthy, pull CloudWatch SuccessfulRequestLatency by Operation (SampleCount = calls/op) and ConsumedReadCapacityUnits to get the real read mix.
---

# Code review can't tell you a DynamoDB design is healthy — measure per-operation

**Problem:** A static audit of the code concludes a DynamoDB table is healthy: every
`Query` carries a `KeyConditionExpression` pinning one partition, no application read
degrades to a Scan, GSIs are used well. The design looks clean — but the table is
quietly expensive, and nobody knows why. The audit counted *call sites in the source*,
which measures the **design**, not the **executed traffic**.

**Root Cause:** Grep-level analysis can't see runtime. A single `Scan` call site buried
in a scheduled worker, a pagination loop, or a background sweep executes on a timer and
can dwarf every interactive read combined — while contributing just one line to the
code. Static review over-weights code that runs rarely and under-weights code that runs
on a 5-minute cron. Only runtime metrics reveal the actual operation mix and where the
read capacity goes.

**Solution:** Measure per-operation before concluding anything about a table's health
or cost. Two CloudWatch signals do it:

1. **Operation mix** — `AWS/DynamoDB` `SuccessfulRequestLatency` carries an `Operation`
   dimension; its `SampleCount` statistic is the number of executed calls of that type.
   Loop over the operations to get real call counts:
   ```bash
   START=$(date -u -v-30d +%Y-%m-%dT%H:%M:%SZ); END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
   for OP in GetItem Query Scan PutItem UpdateItem DeleteItem BatchGetItem TransactWriteItems; do
     N=$(aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB \
       --metric-name SuccessfulRequestLatency \
       --dimensions Name=TableName,Value=my-table Name=Operation,Value=$OP \
       --start-time $START --end-time $END --period 2592000 \
       --statistics SampleCount --query 'Datapoints[0].SampleCount' --output text)
     echo "$OP,$N"
   done
   ```
2. **Where the capacity goes** — `ConsumedReadCapacityUnits` (Sum), dimensioned by
   `TableName` and, separately, by `GlobalSecondaryIndexName`, shows how many read units
   each surface actually burns. A tiny table Scanned on a schedule can be ~95% of read
   capacity even at <10% of interactive value.

Then reconcile: if `Scan` is a large share of executed reads or of consumed capacity,
find the caller (usually a scheduled worker) and fix it (see the TTL/Streams lesson).
The static design being clean does **not** exempt you from this check.

**Key Insight:** Counting query/scan call sites measures your schema's *design*; only
CloudWatch per-`Operation` `SampleCount` + `ConsumedReadCapacityUnits` measure the
*executed traffic*. A DynamoDB table that looks flawless in code review can still spend
most of its read capacity on one scheduled Scan — so verify health from the metrics, not
the source.

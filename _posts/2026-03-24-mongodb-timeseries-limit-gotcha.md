---
title: "MongoDB Time Series: The findOne + sort Performance Trap"
date: 2026-03-24T12:00:00+01:00
permalink: mongodb-timeseries-findone-sort-gotcha
description: Why a simple findOne with a sort on a MongoDB time series collection can silently scan gigabytes of data — and what to do about it
categories:
  - blog
tags:
  - MongoDB
  - Performance
  - Time Series
---

MongoDB's time series collections are a great fit for sensor readings, metrics, and any append-heavy workload. But there's a query pattern that looks completely innocent, runs without errors, and can silently take 10–100× longer than expected: `findOne` with a `sort`.

## The query that looks fine

```js
db.measurements.findOne(
  { "metadata.device_id": "device-42", "metadata.type": "temperature" },
  { sort: { timestamp: -1 } }
)
```

This returns the most recent temperature reading for a device. It uses an index. `explain()` shows an IXSCAN, not a COLLSCAN. It looks correct. And yet on a collection with a few years of data it can take seconds rather than milliseconds.

## What's actually happening

Time series collections don't store documents the way regular collections do. Internally, MongoDB groups measurements into **buckets** — compressed chunks of documents sharing the same metadata, typically spanning an hour each (`bucketMaxSpanSeconds: 3600` by default). What you query as `measurements` is a view; the real storage is `system.buckets.measurements`.

When you run `findOne` with `sort: { timestamp: -1 }` and `limit: 1`, MongoDB rewrites it into an aggregation pipeline:

```
$cursor          → scan system.buckets.measurements
$_internalUnpackBucket → decompress individual measurements
$_internalBoundedSort  → sort and apply limit: 1
$project
```

The index on `(metadata.device_id, metadata.type, timestamp)` correctly narrows the bucket scan to only buckets belonging to that device and type. The IXSCAN is fast. But here is the trap: the **limit applies to individual measurements, not buckets**. The cursor has no way to translate "give me 1 measurement" into "give me N buckets" without first knowing how many measurements are inside each bucket. So it fetches every matching bucket from storage.

If a device has 2 years of hourly buckets, that's ~17,500 buckets. The IXSCAN completes in a few milliseconds. Then those 17,500 bucket documents are read from WiredTiger — and that FETCH is where your seconds disappear.

The `$_internalBoundedSort` optimization does help somewhat: it can stop unpacking once it has found enough measurements to satisfy the sort and limit. In practice you'll often see a small `nReturned` at the pipeline level. But the FETCH stage has already done the damage before the pipeline gets a chance to apply backpressure.

Here's what an `explain("executionStats")` looks like on a collection with 798 matching buckets:

```
IXSCAN   keysExamined: 798   executionTimeMillisEstimate: 5ms
FETCH    docsExamined: 798   executionTimeMillisEstimate: 1,141ms
```

Five milliseconds to scan the index. 1,141 milliseconds to read the documents. The index is doing its job; the bucket architecture is the bottleneck.

## Why a time range filter fixes it

Adding a `timestamp` range to the match narrows the bucket scan before anything is fetched:

```js
const ten_days_ago = new Date(Date.now() - 10 * 24 * 60 * 60 * 1000);

db.measurements.findOne(
  {
    "metadata.device_id": "device-42",
    "metadata.type": "temperature",
    timestamp: { $gte: ten_days_ago }
  },
  { sort: { timestamp: -1 } }
)
```

The index on `(metadata.device_id, metadata.type, control.max.timestamp)` can now skip buckets whose max timestamp falls outside the range. Instead of fetching 798 buckets for a device's entire history, you fetch perhaps 7 — one per day in the last 10 days. The query goes from seconds to sub-millisecond.

## A progressive fallback strategy

The challenge is that you don't always know how far back to look. A device might have been inactive for weeks. One approach is to search backwards through widening windows and stop as soon as you find a result:

```js
async function findLatest(collection, deviceId, type, cutoff) {
  const windows = [
    { start: daysAgo(cutoff, 10),  end: cutoff },
    { start: daysAgo(cutoff, 20),  end: daysAgo(cutoff, 10) },
    { start: daysAgo(cutoff, 40),  end: daysAgo(cutoff, 20) },
    { start: daysAgo(cutoff, 80),  end: daysAgo(cutoff, 40) },
    { start: new Date(0),           end: daysAgo(cutoff, 80) },
  ];

  for (const window of windows) {
    const doc = await collection.findOne(
      {
        "metadata.device_id": deviceId,
        "metadata.type": type,
        timestamp: { $gte: window.start, $lt: window.end },
      },
      { sort: { timestamp: -1 }, projection: { value: 1, _id: 0 } }
    );
    if (doc) return doc;
  }
  return null;
}
```

For an active device the first window hits immediately. Only inactive devices fall through to wider windows, and each fallback query still scans a bounded number of buckets rather than everything.

## Other approaches

**Maintain a "latest value" document.** If you query the most recent reading frequently, write it to a regular collection on each upsert. `findOne` with a point-lookup on an indexed field is O(1) regardless of history depth.

**Use `$group` with `$first`/`$last` in an aggregation.** If you need first and last values for many devices in one call, a single aggregation with `$group` is cheaper than N × 2 individual queries, especially once you factor in round-trip overhead.

**Add a time range to the initial query.** If your use case has a natural time horizon (a report that only covers the last 90 days, for example), apply it upfront rather than relying on a fallback ladder.

## The mental model to keep

With a regular collection, `limit: 1` after an IXSCAN is O(1): the index points directly to the document, one fetch, done. With a time series collection, `limit: 1` after an IXSCAN is O(buckets): every matching bucket must be fetched before the pipeline can apply the limit. The storage and retrieval unit is the bucket, not the measurement. Until you internalize that distinction, time series queries that look indexed and fast can silently cost you orders of magnitude more than expected.

---
title: "How columnar storage actually works (and why ClickHouse is fast)"
date: 2026-03-04
description: "Why ClickHouse queries billions of rows in milliseconds — columnar storage, compression, and the MergeTree engine explained from first principles."
tags: ["databases", "ClickHouse", "system design"]
draft: false
---

Most databases store data on disk. The fundamental question is *how* — and the answer determines whether your analytical query takes 50 milliseconds or 50 seconds.

## The row store model

Traditional databases like PostgreSQL and MySQL store data row by row. Each row is a contiguous block on disk: all columns for a single record live together. This is excellent for transactional workloads — when you `SELECT * FROM users WHERE id = 42`, the database reads one block and gets everything.

But what happens when you run `SELECT AVG(gpu_temp) FROM telemetry WHERE timestamp > now() - INTERVAL 1 HOUR`? You only need one column, but the database reads *all* of them. Every device_id, every event_type, every field you don't care about — all pulled from disk just to reach the temperatures.

```
Row 1: dev_001 | heartbeat | 14:00:01 | 72.5 | 45.2
Row 2: dev_001 | heartbeat | 14:00:02 | 73.1 | 44.8
Row 3: dev_002 | error     | 14:00:01 | 91.3 | 88.7
```

> Imagine a filing cabinet where each drawer holds one complete employee record — name, salary, department, phone, address. To calculate average salary across 10,000 employees, you'd have to open every single drawer and ignore everything except the salary field.

## The column store model

ClickHouse flips this on its head. Each column is stored in its own file:

```
gpu_temp.bin   [72.1, 68.4, 91.2, 73.0, ...]
cpu_usage.bin  [45.2, 38.1, 92.7, 51.3, ...]
device_id.bin  ["d-001", "d-002", "d-001", ...]
```

Now `AVG(gpu_temp)` reads exactly one file. The others aren't touched. For a table with 20 columns, that's a **95% reduction in I/O**.

Row 1 = first entry in all files. Row 2 = second entry. Position is the implicit row identifier.

## Why compression works better

Columnar storage unlocks compression ratios that row stores can't touch. When all values in a file share a type and statistical distribution, compression algorithms feast:

**Dictionary encoding** — a column with 50 distinct device types stores each string once, then uses integer references. A 40-byte string becomes a 2-byte pointer. 50-100x compression.

**Delta encoding** — sorted timestamps often differ by small, predictable amounts. Store the first value, then just the deltas: `[1709571600, +1, +1, +1, +2, +1]`. 5-10x compression.

**Run-length encoding** — if your data is sorted by `device_id`, a million consecutive rows with the same value compress to: `("d-001", 1000000)`. 20-50x compression.

Row-based storage tries to compress mixed types (string, float, int, string...) — compressors can't find patterns. Typical: 2-4x. Columnar: **10-50x**.

In ClickHouse, [`LowCardinality(String)`](https://clickhouse.com/docs/en/sql-reference/data-types/lowcardinality) explicitly tells the engine to use dictionary encoding. Use it for columns with few distinct values: event_type, region, device_type.

## How ClickHouse organizes data on disk

Columnar layout explains the read path, but ClickHouse also needs to *write* fast — it ingests millions of rows per second. This is where the [MergeTree engine](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree) comes in.

Data arrives in **parts**: each insert creates a new immutable directory on disk containing one file per column, pre-sorted by the primary key. Background merges periodically combine smaller parts into larger ones — similar to LSM trees in LevelDB or RocksDB.

The primary key isn't a unique constraint. It's a **sort order**. If you define `ORDER BY (device_id, event_timestamp)`, every part stores rows sorted that way. This has two consequences:

1. **Sparse index.** ClickHouse stores the primary key value for every 8,192nd row (a "granule"). To find rows for `device_id = 'd-001'`, it binary-searches the sparse index, finds which granules match, and reads only those. Entire blocks of irrelevant data get skipped without touching disk.

2. **Sort order drives compression.** Sorted data means identical `device_id` values sit next to each other — run-length encoding kicks in. Timestamps within a device are sequential — delta encoding kicks in. The choice of `ORDER BY` directly determines how compressible your data is.

Getting `ORDER BY` right is the single most impactful decision when designing a ClickHouse table. Wrong sort order means full scans and poor compression. Right sort order means your queries touch 1% of the data.

## The tradeoff: updates and deletes

Columnar storage optimizes for reads at the cost of mutations. Updating a single field in a row store means rewriting one block. In a column store, that same update touches every column file for that row — and in ClickHouse, `ALTER TABLE UPDATE` isn't an in-place operation. It rewrites entire parts in the background, which can take minutes on large tables and spike I/O.

Deletes work the same way. ClickHouse marks rows as deleted and physically removes them during the next merge. Until then, every query still filters out the dead rows at read time.

This is why ClickHouse isn't a good fit for workloads that need frequent row-level mutations — user profiles, shopping carts, anything transactional. It's built for append-heavy, read-heavy patterns: logs, events, telemetry, analytics. If you find yourself writing `ALTER TABLE UPDATE` regularly, you're fighting the storage engine.

## When to use which

**Row-based wins when:**
- Point lookups (`WHERE id = X`) — one contiguous read gets all columns
- Full-row inserts — append a single chunk
- Row-level updates — rewrite in place
- OLTP workloads (web apps, banking)

**Column-based wins when:**
- Aggregates (`AVG`, `SUM`, `COUNT`) — reads only needed columns
- Wide table scans (need 3 of 20 columns) — skips 85%+ of I/O
- Compression matters — 10-50x vs 2-4x
- Time-series and telemetry — scan billions of rows, few columns

This is why the database world is split: PostgreSQL/MySQL for transactional workloads, ClickHouse/BigQuery for analytical workloads. Not because one is better — because they solve fundamentally different problems.

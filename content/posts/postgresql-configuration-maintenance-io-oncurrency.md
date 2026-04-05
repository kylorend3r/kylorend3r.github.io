+++
title = 'Maintenance IO Concurrency under the Hoods in PostgreSQL'
date = '2026-02-08'
draft = false
description = 'A deep dive into maintenance_io_concurrency — how PostgreSQL uses it during VACUUM and index builds, how it differs from effective_io_concurrency, and how to tune it for your storage'
tags = ['postgresql', 'vacuum', 'performance', 'configuration', 'database', 'maintenance', 'io']
author = 'kylorend3r'
+++



## Introduction

In the world of PostgreSQL performance tuning, we often obsess over query execution times. We tweak indexes, rewrite joins, and adjust work_mem. However, a database is only as fast as its ability to maintain itself. This is where maintenance_io_concurrency comes into play. Introduced to allow PostgreSQL to take advantage of modern SSDs and RAID arrays during heavy lifting, this Grand Unified Configuration (GUC) setting controls the number of concurrent I/O operations that PostgreSQL can have "in flight" during maintenance tasks.

Why do we need to consider the `maintenance_io_concurrency` GUC in PostgreSQL ? Do we need to consider it right after create our database cluster ? In this blopost, I'll try to discuss how PostgreSQL uses the GUC and how can we gain more benefits by tuning it wisely. 


## Table of Contents

- [Beyond the Surface](#beyond-the-surface)
- [Which Operations Actually Use It?](#which-operations-actually-use-it)
- [effective_io_concurrency vs maintenance_io_concurrency](#effective_io_concurrency-vs-maintenance_io_concurrency)
- [Tablespace-Level Configuration](#tablespace-level-configuration)
- [Observing the Effect](#observing-the-effect)
- [Tuning Recommendations](#tuning-recommendations)
- [Conclusion](#conclusion)

## Beyond the Surface

To truly understand this GUC, we have to look past the name and into the source code. At its core, maintenance_io_concurrency is designed to improve read streaming operations.

When PostgreSQL performs maintenance, it doesn't just read one block at a time and wait; it attempts to "look ahead" and pull data into the buffer cache before it’s actually needed. 

During a VACUUM (especially the parallel variety), the system needs to scan heaps and indexes to identify dead tuples. By increasing the I/O concurrency, Postgres can trigger asynchronous I/O requests to the OS, ensuring that by the time the CPU is ready to process the next page, that page is already sitting in memory.

```c
	/*
	 * Decide how many I/Os we will allow to run at the same time.  That
	 * currently means advice to the kernel to tell it that we will soon read.
	 * This number also affects how far we look ahead for opportunities to
	 * start more I/Os.
	 */
	tablespace_id = smgr->smgr_rlocator.locator.spcOid;
	if (!OidIsValid(MyDatabaseId) ||
		(rel && IsCatalogRelation(rel)) ||
		IsCatalogRelationOid(smgr->smgr_rlocator.locator.relNumber))
	{
		/*
		 * Avoid circularity while trying to look up tablespace settings or
		 * before spccache.c is ready.
		 */
		max_ios = effective_io_concurrency;
	}
	else if (flags & READ_STREAM_MAINTENANCE)
		max_ios = get_tablespace_maintenance_io_concurrency(tablespace_id);
	else
		max_ios = get_tablespace_io_concurrency(tablespace_id);

	/* Cap to INT16_MAX to avoid overflowing below */
	max_ios = Min(max_ios, PG_INT16_MAX);
```

This snippet is the decision gate inside PostgreSQL's read-stream infrastructure. Three branches, three outcomes. If the relation is a catalog or the database OID is not yet set, it falls back to `effective_io_concurrency`. If the `READ_STREAM_MAINTENANCE` flag is set, it uses `maintenance_io_concurrency` (resolved per tablespace). Otherwise — for a regular user query — it uses `effective_io_concurrency`. This is why the two parameters behave independently: the flag on the read stream, not the caller, determines which limit applies.

The "look-ahead" mechanism itself works by issuing `posix_fadvise(POSIX_FADV_WILLNEED)` calls for upcoming blocks while the current block is being processed. The kernel takes the hint and starts prefetching those pages from disk into the OS page cache. By the time PostgreSQL's backend arrives at those pages, they are already warm. On a spinning disk the gain can be dramatic. On NVMe storage it is less pronounced — the access latency is so low that the head-start matters less — but the parallelism it enables across multiple I/O queues still helps.

## Which Operations Actually Use It?

The `READ_STREAM_MAINTENANCE` flag is set by the code paths responsible for heavy, background-style reads. In practice this covers the following operations:

**VACUUM and autovacuum** are the most common consumer. Both the heap scan and the index vacuuming passes use a read stream, and both pass the maintenance flag. This means every autovacuum worker on your system is throttled by this parameter. If you have a busy database with aggressive autovacuum, tuning `maintenance_io_concurrency` is at least as important as tuning `autovacuum_vacuum_cost_delay`.

**`CREATE INDEX`** uses a read stream when scanning the heap to build the index. The same is true for `REINDEX`. In both cases the heap scan is a sequential forward pass over potentially gigabytes of data — exactly the workload that benefits from read-ahead.

**`CLUSTER`** rewrites the entire heap in index order and also scans the old heap as an input. Both the read and the rewrite benefit from higher concurrency.

**`ANALYZE`** reads a sample of blocks from each table. While the sample is not strictly sequential, the read-stream infrastructure is still used and still respects the maintenance limit.

It is worth noting that none of the above are user-facing query operations. They run in the background, in maintenance windows, or in autovacuum workers. This separation is intentional — you can tune maintenance aggressiveness without touching the read concurrency available to your regular queries.

## effective_io_concurrency vs maintenance_io_concurrency

These two parameters are often confused because their names are similar and they both live in `postgresql.conf` side by side. The distinction is clean once you know where the flag is set:

| | `effective_io_concurrency` | `maintenance_io_concurrency` |
|---|---|---|
| **Governs** | User query read streams | Maintenance task read streams |
| **Used by** | Bitmap heap scans, parallel seq scans | VACUUM, CREATE INDEX, CLUSTER, ANALYZE |
| **Default** | 16 (PostgreSQL 16+) | 10 |
| **Scope** | Session-level (can be SET per session) | Session-level (can be SET per session) |
| **Tablespace override** | Yes | Yes |

`effective_io_concurrency` was the original parameter introduced in PostgreSQL 9.1. `maintenance_io_concurrency` was added later (PostgreSQL 13) specifically to decouple maintenance throughput from query throughput. Before that, a DBA tuning for fast VACUUM on SSDs had no choice but to raise `effective_io_concurrency`, which also affected all user queries. Now the two can be set independently.

A practical example: on a system where your user queries are latency-sensitive and you do not want extra prefetch adding jitter, you might keep `effective_io_concurrency` at 16 but set `maintenance_io_concurrency` to 128, giving VACUUM and `CREATE INDEX` the benefit of the fast NVMe storage without affecting query behaviour at all.

```sql
-- check current values
SHOW effective_io_concurrency;
SHOW maintenance_io_concurrency;

-- set them independently
ALTER SYSTEM SET effective_io_concurrency = 16;
ALTER SYSTEM SET maintenance_io_concurrency = 128;
SELECT pg_reload_conf();
```

Both parameters accept `0` as a valid value, which disables asynchronous prefetch entirely and forces purely synchronous sequential reads. This is occasionally useful as a diagnostic baseline — if turning a parameter to `0` makes no difference in throughput, your storage is fast enough that prefetch adds no value.

## Tablespace-Level Configuration

One of the more powerful and underused features is that both `effective_io_concurrency` and `maintenance_io_concurrency` can be overridden at the tablespace level. This matters when your cluster has mixed storage — for example, a fast NVMe tablespace for active tables and a slower HDD tablespace for archives.

```sql
-- Override for a fast NVMe tablespace
ALTER TABLESPACE fast_nvme_ts
  SET (effective_io_concurrency = 256, maintenance_io_concurrency = 256);

-- Conservative override for a HDD-backed archive tablespace
ALTER TABLESPACE archive_ts
  SET (effective_io_concurrency = 2, maintenance_io_concurrency = 2);
```

When a read stream is created for a relation, PostgreSQL resolves the tablespace the relation lives in and calls `get_tablespace_maintenance_io_concurrency()` — which is exactly what the source code snippet earlier shows. If no tablespace-level override is set, it falls back to the cluster-wide GUC. This means you can have a single PostgreSQL instance with radically different I/O concurrency profiles per storage tier without any session-level management.

To inspect current tablespace-level overrides:

```sql
SELECT spcname, spcoptions
FROM pg_tablespace
WHERE spcoptions IS NOT NULL;
```

## Observing the Effect

Before tuning any GUC it helps to understand whether the current value is a bottleneck. There are a few views worth querying.

**`pg_stat_progress_vacuum`** shows the live state of any running VACUUM. The `heap_blks_scanned` and `heap_blks_vacuumed` counters give you throughput over time. If the vacuum rate is flat and your storage has headroom, raising `maintenance_io_concurrency` is a reasonable next step.

```sql
SELECT relid::regclass AS table,
       phase,
       heap_blks_total,
       heap_blks_scanned,
       heap_blks_vacuumed,
       index_vacuum_count
FROM pg_stat_progress_vacuum;
```

**`pg_stat_io`**, available since PostgreSQL 16, breaks down I/O by backend type and context. The `context` column distinguishes `vacuum`, `normal`, and `bulkread` — so you can directly observe how many read operations maintenance tasks are generating compared to regular queries.

```sql
SELECT backend_type,
       context,
       reads,
       read_time,
       writes,
       write_time
FROM pg_stat_io
WHERE backend_type IN ('autovacuum worker', 'client backend')
ORDER BY backend_type, context;
```

If `read_time` for autovacuum workers is high relative to `reads`, the workers are waiting for I/O — a sign that the current concurrency setting is too conservative for your storage. If `reads` is very high and `read_time` per read is already low, raising the concurrency further will have diminishing returns.

## Tuning Recommendations

There is no universal correct value for `maintenance_io_concurrency` — it depends almost entirely on your storage subsystem. The following table gives reasonable starting points:

| Storage type | Recommended value |
|---|---|
| Single HDD | 1–4 |
| RAID of HDDs | 4–8 |
| SATA SSD | 32–64 |
| NVMe SSD | 128–256 |
| Cloud block storage (gp2/gp3) | 16–64 (test per IOPS quota) |

The reason HDDs sit so low is that they have a single read head and queuing more I/O requests causes seek contention rather than parallelism. For SSDs and NVMe, the storage controller can genuinely service many concurrent requests and the prefetch gains compound.

When testing, a practical approach is to run a controlled `VACUUM VERBOSE` on a large table while monitoring wall-clock duration:

```sql
-- Establish a baseline
SET maintenance_io_concurrency = 10;
VACUUM VERBOSE large_table;

-- Try a higher value
SET maintenance_io_concurrency = 128;
VACUUM VERBOSE large_table;
```

Keep in mind that `SET` changes the value only for the current session, so you can experiment without touching `postgresql.conf`. Once you find a value that improves throughput without saturating storage, apply it with `ALTER SYSTEM` and reload.

One thing to keep in mind: if you are running `pg_upgrade` or doing a large initial `COPY` load, these operations also benefit from a temporarily higher `maintenance_io_concurrency`. It is reasonable to raise it aggressively during a maintenance window and bring it back to the steady-state value afterwards.

## Conclusion

`maintenance_io_concurrency` is a quiet but impactful parameter. It does not affect your queries directly, but it governs how fast VACUUM reclaims dead tuples, how quickly `CREATE INDEX` builds new indexes, and how efficiently `CLUSTER` rewrites your tables. On modern SSD hardware its default value of 10 is almost certainly leaving throughput on the table.

The key takeaway is that it is independent of `effective_io_concurrency` by design. Tune each one for its own audience — queries on one side, maintenance on the other — and use tablespace-level overrides if your storage topology is not uniform. Before changing anything in production, use `pg_stat_progress_vacuum` and `pg_stat_io` to confirm that I/O is actually the limiting factor.

Demir.
+++
title = 'Optimizing PostgreSQL Tables: Exploring Vacuuming Strategies to Combat Bloat'
date = '2024-04-08'
draft = false
description = 'Understanding PostgreSQL bloat, its impact on performance, and exploring vacuuming strategies including VACUUM FULL and pg_repack to combat table and index bloat'
tags = ['postgresql', 'bloat', 'vacuum', 'database', 'performance', 'maintenance', 'pg_repack']
author = 'kylorend3r'
+++

## Introduction

When it comes to managing a PostgreSQL database, one of the key concerns that database administrators face is bloat. Bloat refers to the unnecessary growth of database objects, such as tables and indexes, leading to inefficient storage utilization and degraded performance. In this article, we will jump into what bloat is, why it matters, and how it can be increased.

## Table of Contents

* [What is Bloat in PostgreSQL?](#what-is-bloat-in-postgresql)
* [Why Does Bloat Matter?](#why-does-bloat-matter)
* [Factors Contributing to Bloat](#factors-contributing-to-bloat)
* [How to Analyze Bloat in Tables](#how-to-analyze-bloat-in-tables)
* [Cleaning Bloat](#cleaning-bloat)
  * [Vacuum](#vacuum)
  * [Pg_Repack](#pg_repack)
* [Conclusion](#conclusion)

## What is Bloat in PostgreSQL?

In PostgreSQL, bloat occurs when database objects occupy more space than is necessary for storing the actual data. This excess space consumption can result from various factors such as outdated statistics, frequent updates and deletions, or inefficient maintenance operations.

There are primarily two types of bloat that can affect PostgreSQL databases:

**Table Bloat:** This occurs when the actual size of a table on disk exceeds the size that it should ideally occupy. Table bloat often arises from frequent updates and deletions, causing the table to become fragmented and inefficiently utilize disk space.

**Index Bloat:** Indexes in PostgreSQL are essential for optimizing query performance. However, index bloat occurs when the size of an index becomes larger than necessary due to redundant or outdated entries. This can lead to slower query execution times and increased disk I/O.

## Why Does Bloat Matter?

Bloat can have several adverse effects on the performance and stability of a PostgreSQL database:

**Reduced Performance:** Bloated tables and indexes can result in slower query execution times as the database engine needs to process unnecessary data. This can lead to degraded application performance and user experience.

**Increased Storage Costs:** Inefficient storage utilization due to bloat can lead to increased storage costs, especially in cloud environments where storage resources are billed based on usage.

**Maintenance Overhead:** Managing a bloated database requires additional maintenance efforts, including routine vacuuming and reindexing operations. Failure to address bloat in a timely manner can lead to further degradation of performance and stability.

## Factors Contributing to Bloat

Several factors can contribute to the increase in bloat ratio within a PostgreSQL database:

**High Update/Delete Activity:** Tables and indexes that undergo frequent updates and deletions are more prone to bloat. Each modification operation can leave behind dead tuples (unused space) that contribute to bloat over time.

**Missing or Outdated Statistics:** Inaccurate statistics can mislead the query planner, leading to suboptimal execution plans and increased bloat. Regularly updating statistics using the `ANALYZE` command is crucial for maintaining optimal performance.

**Long-Running Transactions:** Transactions that hold locks for an extended period can prevent vacuuming processes from reclaiming space efficiently, leading to bloated tables and indexes.

**Inefficient Maintenance Practices:** Failure to perform regular vacuuming, reindexing, and maintenance tasks can allow bloat to accumulate unchecked, resulting in degraded performance and stability over time.

## How to Analyze Bloat in Tables

To analyze bloat in tables we can use the following SQL snippet.

```sql
SELECT current_database(), schemaname, tblname, bs*tblpages AS real_size,
  (tblpages-est_tblpages)*bs AS extra_size,
  CASE WHEN tblpages > 0 AND tblpages - est_tblpages > 0
    THEN 100 * (tblpages - est_tblpages)/tblpages::float
    ELSE 0
  END AS extra_pct, fillfactor,
  CASE WHEN tblpages - est_tblpages_ff > 0
    THEN (tblpages-est_tblpages_ff)*bs
    ELSE 0
  END AS bloat_size,
  CASE WHEN tblpages > 0 AND tblpages - est_tblpages_ff > 0
    THEN 100 * (tblpages - est_tblpages_ff)/tblpages::float
    ELSE 0
  END AS bloat_pct, is_na
  -- , tpl_hdr_size, tpl_data_size, (pst).free_percent + (pst).dead_tuple_percent AS real_frag -- (DEBUG INFO)
FROM (
  SELECT ceil( reltuples / ( (bs-page_hdr)/tpl_size ) ) + ceil( toasttuples / 4 ) AS est_tblpages,
    ceil( reltuples / ( (bs-page_hdr)*fillfactor/(tpl_size*100) ) ) + ceil( toasttuples / 4 ) AS est_tblpages_ff,
    tblpages, fillfactor, bs, tblid, schemaname, tblname, heappages, toastpages, is_na
    -- , tpl_hdr_size, tpl_data_size, pgstattuple(tblid) AS pst -- (DEBUG INFO)
  FROM (
    SELECT
      ( 4 + tpl_hdr_size + tpl_data_size + (2*ma)
        - CASE WHEN tpl_hdr_size%ma = 0 THEN ma ELSE tpl_hdr_size%ma END
        - CASE WHEN ceil(tpl_data_size)::int%ma = 0 THEN ma ELSE ceil(tpl_data_size)::int%ma END
      ) AS tpl_size, bs - page_hdr AS size_per_block, (heappages + toastpages) AS tblpages, heappages,
      toastpages, reltuples, toasttuples, bs, page_hdr, tblid, schemaname, tblname, fillfactor, is_na
      -- , tpl_hdr_size, tpl_data_size
    FROM (
      SELECT
        tbl.oid AS tblid, ns.nspname AS schemaname, tbl.relname AS tblname, tbl.reltuples,
        tbl.relpages AS heappages, coalesce(toast.relpages, 0) AS toastpages,
        coalesce(toast.reltuples, 0) AS toasttuples,
        coalesce(substring(
          array_to_string(tbl.reloptions, ' ')
          FROM 'fillfactor=([0-9]+)')::smallint, 100) AS fillfactor,
        current_setting('block_size')::numeric AS bs,
        CASE WHEN version()~'mingw32' OR version()~'64-bit|x86_64|ppc64|ia64|amd64' THEN 8 ELSE 4 END AS ma,
        24 AS page_hdr,
        23 + CASE WHEN MAX(coalesce(s.null_frac,0)) > 0 THEN ( 7 + count(s.attname) ) / 8 ELSE 0::int END
           + CASE WHEN bool_or(att.attname = 'oid' and att.attnum < 0) THEN 4 ELSE 0 END AS tpl_hdr_size,
        sum( (1-coalesce(s.null_frac, 0)) * coalesce(s.avg_width, 0) ) AS tpl_data_size,
        bool_or(att.atttypid = 'pg_catalog.name'::regtype)
          OR sum(CASE WHEN att.attnum > 0 THEN 1 ELSE 0 END) <> count(s.attname) AS is_na
      FROM pg_attribute AS att
        JOIN pg_class AS tbl ON att.attrelid = tbl.oid
        JOIN pg_namespace AS ns ON ns.oid = tbl.relnamespace
        LEFT JOIN pg_stats AS s ON s.schemaname=ns.nspname
          AND s.tablename = tbl.relname AND s.inherited=false AND s.attname=att.attname
        LEFT JOIN pg_class AS toast ON tbl.reltoastrelid = toast.oid
      WHERE NOT att.attisdropped
        AND tbl.relkind in ('r','m')
      GROUP BY 1,2,3,4,5,6,7,8,9,10
      ORDER BY 2,3
    ) AS s
  ) AS s2
) AS s3
 WHERE schemaname='public'
ORDER BY schemaname, tblname;
```

Besides the query, we can use `pgstattuple` extension to discover and analyze tuples in the table.

To install the `pgstattuple` extension in your PostgreSQL database, follow these steps:

```sql
CREATE EXTENSION pgstattuple;
```

After that, we can call `pgstattuple` function:

```sql
blog_bench=# SELECT * FROM pgstattuple('test_table');
 table_len | tuple_count | tuple_len | tuple_percent | dead_tuple_count | dead_tuple_len | dead_tuple_percent | free_space | free_percent
-----------+-------------+-----------+---------------+------------------+----------------+--------------------+------------+--------------
 102400000 |     1500000 |  91500000 |         89.36 |                0 |              0 |                  0 |      50000 |         0.05
```

A `dead_tuple_percent` greater than 50% suggests that more than half of the tuples in the table are dead, meaning they are no longer needed by any active transaction. This level of bloat indicates that the table has undergone extensive data modifications (inserts, updates, deletes) without proper vacuuming or maintenance. Dead tuples occupy space within the table's data pages, contributing to wasted storage. This inefficiency can lead to increased disk usage and decreased overall database performance.

The analysis of bloat ratios exceeding 50% in PostgreSQL tables underscores the importance of considering the size of the tables in addition to optimizing autovacuum and analyze settings and data types. Larger tables are more prone to fragmentation and bloat, especially if they undergo frequent updates and deletes. Therefore, meticulous attention to managing bloat becomes even more critical for maintaining the performance and efficiency of sizable tables. By prioritizing bloat management strategies alongside considerations of table size, administrators can effectively mitigate inefficiencies, optimize resource usage, and ensure the smooth operation of their PostgreSQL databases.

* Autovacuum and Analyze settings.
* Column types and lengths

## Cleaning Bloat

### Vacuum

In PostgreSQL, one method to manually clean bloat is by using the `VACUUM FULL` command. Unlike the regular `VACUUM` command, which only reclaims space and marks dead tuples for reuse, `VACUUM FULL` physically rewrites the entire table, reclaiming space and eliminating bloat in the process. However, while effective, it comes with significant drawbacks, particularly in terms of locking and potential downtime for applications.

When `VACUUM FULL` is executed, it acquires an exclusive lock on the table being vacuumed. This means that no other transactions can read from or write to the table until the vacuum operation completes. Depending on the size of the table and the amount of bloat, this lock can be held for an extended period, causing significant downtime for applications that rely on the affected table.

Moreover, since `VACUUM FULL` rewrites the entire table, it can also consume a considerable amount of system resources, including CPU and disk I/O, potentially impacting the performance of other operations running on the database server.

Therefore, while `VACUUM FULL` can effectively clean bloat, it should be used judiciously, particularly in production environments where minimizing downtime is crucial. Alternatives such as `VACUUM` or `VACUUM ANALYZE` with appropriate tuning may be more suitable for routine maintenance tasks, reserving `VACUUM FULL` for situations where significant bloat needs to be urgently addressed, and downtime can be tolerated.

```
 current_database | schemaname |  tblname   | real_size | extra_size | extra_pct | fillfactor | bloat_size | bloat_pct | is_na
------------------+------------+------------+-----------+------------+-----------+------------+------------+-----------+-------
 blog_bloat       | public     | test_table | 256000000 |   51396608 |   20.0768 |        100 |   51396608 |   20.0768 | f
```

```sql
VACUUM FULL test_table;
```

```
 current_database | schemaname |  tblname   | real_size | extra_size | extra_pct | fillfactor | bloat_size | bloat_pct | is_na
------------------+------------+------------+-----------+------------+-----------+------------+------------+-----------+-------
 blog_bloat       | public     | test_table | 204800000 |     196608 |     0.096 |        100 |     196608 |     0.096 | f
```

Using `VACUUM FULL` to clean bloat in a table in PostgreSQL comes with its own set of advantages and disadvantages:

**Advantages:**

* **Comprehensive Space Reclamation:** `VACUUM FULL` physically rewrites the entire table, effectively reclaiming all wasted space and reducing bloat to a minimum. This can result in significant disk space savings compared to other vacuuming methods.
* **Simplicity:** `VACUUM FULL` is a built-in command in PostgreSQL and doesn't require any additional extensions or tools to use. This simplifies the process of reclaiming space and managing bloat.
* **Immediate Impact:** Since `VACUUM FULL` rewrites the entire table in one operation, its effects are immediate. Once the operation completes, the table is compacted, and the reclaimed space becomes available for reuse.

**Disadvantages:**

* **Locking:** One of the most significant drawbacks of `VACUUM FULL` is the explicit locking it imposes on the table being vacuumed. This exclusive lock prevents any other transactions from accessing the table until the vacuum operation completes. This can lead to significant downtime for applications that rely on the affected table, making it unsuitable for use in production environments during peak hours.
* **Resource Intensive:** Since `VACUUM FULL` rewrites the entire table, it can consume a considerable amount of system resources, including CPU, memory, and disk I/O. This can impact the performance of other operations running on the database server during the vacuuming process.
* **Table Bloat Recurrence:** While `VACUUM FULL` effectively reduces bloat at the time of execution, bloat can still accumulate over time as the table continues to undergo updates and deletes. As a result, `VACUUM FULL` may need to be periodically re-run to maintain optimal table efficiency, leading to potential downtime and resource consumption each time it's executed.
* **Impact on Replication:** If the database is part of a replication setup, using `VACUUM FULL` can lead to increased replication lag, as the entire table needs to be rewritten, affecting the replication process's ability to keep up with changes.

In comparison to tools like `pg_repack`, which can perform similar table reorganization tasks without requiring explicit locking, the advantage of `VACUUM FULL` lies in its simplicity and built-in nature. However, the trade-off is the significant downtime and resource consumption it entails during execution. Ultimately, the choice between `VACUUM FULL` and other vacuuming methods depends on the specific requirements and constraints of the PostgreSQL environment, balancing the need for space reclamation with the impact on application availability and server resources.

### Pg_Repack

`pg_repack` is an external extension for PostgreSQL that provides an alternative method for reclaiming space and reducing bloat. Unlike `VACUUM FULL`, `pg_repack` performs an online table repack operation, allowing applications to continue functioning normally during the process. By leveraging advanced techniques, `pg_repack` effectively cleans bloat without explicit table locking, minimizing downtime and resource consumption.

To install `pg_repack` follow these steps:

**Install repack packages on OS:**

```bash
apt-get update -y && apt-get install postgresql-15-repack -y
```

**Create extension:**

```sql
CREATE EXTENSION pg_repack;
```

After installing `pg_repack` you can apply the following bash scripts with postgres user on operating system.

```bash
pg_repack -d blog_bench -t public.test_table --no-kill-backend --jobs 4 --echo
pg_repack -d blog_bench -t public.test_table --no-kill-backend --only-indexes --jobs 4 --echo
```

**Advantages of pg_repack:**

* **Online Operation:** `pg_repack` performs table repacking online, ensuring that applications can continue to read and write to the table without interruption. This minimizes downtime and ensures uninterrupted access to your database.
* **Efficient Bloat Reduction:** `pg_repack` utilizes sophisticated algorithms to efficiently reclaim wasted space within tables, optimizing resource utilization and improving overall database performance.

It's important to note that `VACUUM FULL` and `pg_repack` are not the only methods available for cleaning and managing bloat in PostgreSQL databases. Various other techniques, such as using different vacuuming options (`VACUUM`, `ANALYZE`, etc.), table reorganization, or even database reindexing, can also be employed based on the specific requirements and characteristics of your database environment. The choice of method may vary depending on factors such as table size, update patterns, application workload, and available maintenance windows. However, regardless of the approach chosen, the ultimate goal remains consistent: to keep bloat under control and ensure optimal database performance. By adopting a proactive approach to bloat management and regularly monitoring and adjusting vacuuming strategies as needed, PostgreSQL administrators can effectively maintain database health and performance over time.

## Conclusion

Embrace the Dark Side as you jump into database optimization! Have questions or need guidance on your journey? Reach out to me on LinkedIn, connect on Twitter or Superpeer. May the Force guide us as we optimize our databases for greater efficiency.

## References

* [PostgreSQL Documentation: pgstattuple](https://www.postgresql.org/docs/current/pgstattuple.html)
* [pg_repack Documentation](https://reorg.github.io/pg_repack/)
* [PostgreSQL Documentation: VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html)

Demir.

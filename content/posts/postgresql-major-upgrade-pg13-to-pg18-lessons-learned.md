+++
title = 'What It Actually Takes to Upgrade PostgreSQL in Production Without Breaking Everything'
date = '2026-05-14'
draft = false
description = 'A practical guide to planning and executing a PostgreSQL major version upgrade at scale: why we upgraded, how we designed the playbook, why the process was still dangerous, and two real bugs we discovered, reported to the community, and solved.'
tags = ['postgresql', 'upgrade', 'pg_upgrade', 'vacuum', 'performance', 'database', 'replication', 'pg_init_privs']
author = 'kylorend3r'
+++

## Table of Contents

- [Table of Contents](#table-of-contents)
- [Why We Upgraded](#why-we-upgraded)
- [VACUUM Improvements](#vacuum-improvements)
  - [Bottom-Up Index Deletion](#bottom-up-index-deletion)
  - [compactify\_tuples Performance](#compactify_tuples-performance)
  - [Radix Tree TID Store (PostgreSQL 17+)](#radix-tree-tid-store-postgresql-17)
  - [Asynchronous I/O for Sequential Scans](#asynchronous-io-for-sequential-scans)
- [WAL Insert Lock Optimization](#wal-insert-lock-optimization)
- [Logical Replication Improvements](#logical-replication-improvements)
- [Monitoring Improvements](#monitoring-improvements)
- [What We Considered Before Upgrading](#what-we-considered-before-upgrading)
- [Choosing the Target Version: PostgreSQL 17 or 18](#choosing-the-target-version-postgresql-17-or-18)
- [The Upgrade Playbook Design](#the-upgrade-playbook-design)
  - [Reducing Upgrade Duration](#reducing-upgrade-duration)
- [Preflight: The Most Important Phase](#preflight-the-most-important-phase)
- [Why Upgrade Process Was Dangerous and Risky Still](#why-upgrade-process-was-dangerous-and-risky-still)
  - [Logical Replication: Two Scenarios](#logical-replication-two-scenarios)
  - [The Constraint That Existed Twice](#the-constraint-that-existed-twice)
    - [The Root Cause](#the-root-cause)
    - [The Scenario That Triggers It](#the-scenario-that-triggers-it)
    - [The Fix](#the-fix)
    - [Community Bug Report](#community-bug-report)
    - [Summary for This Problem](#summary-for-this-problem)
  - [The Role That Was Gone But Not Forgotten](#the-role-that-was-gone-but-not-forgotten)
    - [Background: What is pg\_init\_privs](#background-what-is-pg_init_privs)
    - [The Sequence That Produces Orphan OIDs](#the-sequence-that-produces-orphan-oids)
    - [How This Breaks pg\_upgrade](#how-this-breaks-pg_upgrade)
    - [The Fix: Clean Up Orphan OIDs Before Upgrading](#the-fix-clean-up-orphan-oids-before-upgrading)
- [Post-Upgrade Metrics and Validation](#post-upgrade-metrics-and-validation)
  - [Log Prerequisites](#log-prerequisites)
  - [Error and Fatal Rate](#error-and-fatal-rate)
  - [Slow Query Volume](#slow-query-volume)
  - [Cancelled Queries](#cancelled-queries)
  - [Checkpoint Duration](#checkpoint-duration)
  - [Autovacuum Duration](#autovacuum-duration)
- [Conclusion: A Checklist for DBAs](#conclusion-a-checklist-for-dbas)

---

## Why We Upgraded

PostgreSQL major version upgrades are rarely simple. The larger the cluster — more databases, more extensions, more replication topology — the more moving parts you have to coordinate simultaneously. This post documents our journey upgrading a production PostgreSQL infrastructure from version 13 to version 18. It covers the reasons that drove the decision, the automated playbook we designed to handle it, the edge cases we discovered along the way, and most importantly, two non-obvious bugs we encountered and had to solve before the upgrade could succeed.

The goal is not just to share what happened to us — it is to give you a concrete checklist so that your upgrade does not fail on something that ours already did.

PostgreSQL 13 served us well, but over time several areas started showing their age under production load. Four stood out: autovacuum performance, WAL write throughput under concurrent load, logical replication maturity, and observability. All four are areas where PostgreSQL 14 through 18 made significant improvements that directly impacted our operational stability. The sections below cover each in detail.

---

## VACUUM Improvements

### Bottom-Up Index Deletion

PostgreSQL 14 introduced `bottom-up index deletion` for B-tree indexes. Before this change, heavy `UPDATE` workloads on tables with indexed columns caused significant index bloat. Every update creates a new heap tuple version and a new index entry pointing to it. Under high-update traffic, index pages would fill up with stale entries pointing to dead heap tuples, leading to page splits and growing index size.

Bottom-up deletion changes this. Instead of splitting the page when it is full, the running process first scans the current index leaf page for older entries pointing to dead heap tuples. It removes those stale entries in-place to make room for the new entry, often avoiding the page split entirely. The result is less bloat and less index maintenance overhead.

### compactify_tuples Performance

The `compactify_tuples` function is used by both VACUUM and autovacuum after removing dead tuples from a heap page. In PostgreSQL 14 this function was optimized, and the improvement flows directly into vacuum performance. Benchmarks show VACUUM running approximately 25% faster on the same workload — from 4.1 seconds down to 2.9 seconds for a heavy-update test table.

### Radix Tree TID Store (PostgreSQL 17+)

This is one of the most impactful improvements for autovacuum-heavy environments. PostgreSQL 17 replaced the internal array used to track dead tuple IDs (TIDs) during vacuum with a Radix Tree implementation. The problem with the old array was capacity: once autovacuum exhausted the memory budget tracked by `maintenance_work_mem` or `autovacuum_work_mem`, it had to stop, process all indexes, and then resume heap scanning. This cycle repeated multiple times on large tables.

A Radix Tree is dramatically more memory-efficient for sparse TID sets. The same memory budget now holds far more TIDs, meaning autovacuum can complete a heap pass in fewer index vacuum rounds. The practical result is that autovacuum runs shorter and touches I/O less frequently.

We benchmarked this directly. A 50-million-row table was created on both PostgreSQL 12 and PostgreSQL 17, an `UPDATE` was issued to touch every row, and autovacuum was monitored via `pg_stat_progress_vacuum`:

```sql
CREATE TABLE test AS SELECT * FROM generate_series(1, 50000000) x(id);
CREATE INDEX ON test(id);
UPDATE test SET id = id - 1;
SELECT * FROM pg_stat_progress_vacuum \watch
```

On PostgreSQL 13, the `index_vacuum_count` reached 3 by the time the heap was only half-scanned — the TID array had filled up and forced multiple index passes. On PostgreSQL 17, the entire 50-million-row table was processed in a single index vacuum round.

![pg_stat_progress_vacuum live monitoring — PG12 vs PG17 during the 50M-row benchmark](/images/posts/postgresql-major-upgrade-pg13-to-pg18-lessons-learned/Screenshot%202024-10-28%20at%2008.08.14.png)

The overall conclusion from our benchmarks: autovacuum on PostgreSQL 12/13 is at least 4 times slower than on PostgreSQL 17 for tables with high dead tuple density. This directly translates to less I/O pressure, lower CPU usage during maintenance windows, and reduced risk of autovacuum falling behind on large tables.

![Autovacuum duration comparison: PostgreSQL 12 (~950s) vs PostgreSQL 17 (~225s)](/images/posts/postgresql-major-upgrade-pg13-to-pg18-lessons-learned/upgrade_vacuum_benchmark.png)

### Asynchronous I/O for Sequential Scans

PostgreSQL 18 introduces an asynchronous I/O subsystem that benefits sequential scans, which VACUUM relies on heavily for heap scanning. By reducing I/O wait time during the heap scan phase, the entire vacuum cycle completes faster, particularly on storage with high latency.

---

## WAL Insert Lock Optimization

On high-core-count systems with concurrent write workloads, PostgreSQL 13 and earlier had a scalability ceiling caused by `WALInsertLock`. When multiple backends concurrently wrote WAL records, they contended on an exclusive lightweight lock to reserve space in the WAL buffer and copy their record. Only one backend could proceed at a time. On systems with many CPU cores and hundreds of concurrent transactions, this lock became a significant bottleneck that capped transactions per second regardless of available hardware.

PostgreSQL 14 replaced this lock with atomic CPU operations (compare-and-swap instructions) for the common case of reserving WAL buffer space. Multiple backends can now reserve space simultaneously. The lock is only acquired for infrequent heavy operations like buffer flush or wrap-around. The result is that WAL write throughput scales much more linearly with CPU core count.

Our pgbench benchmark across 100 concurrent clients and 4 threads made this concrete:

```text
PostgreSQL 13 — 100 clients, 4 threads
TPS:              10,426
Average latency:  9.591 ms
Total transactions: 1,252,152

PostgreSQL 17 — 100 clients, 4 threads
TPS:              12,713
Average latency:  7.866 ms
Total transactions: 1,526,517
Failed:           0
```

A ~22% TPS improvement and ~18% latency reduction under the same concurrent load. PostgreSQL 13 has a lower ceiling for concurrent writes — a scalability risk that compounds as transaction volume grows.

PS: The following benchmark results rely on pgbench outputs. Within a real production environment the results can differ — better or worse depending on workload. Therefore, do not treat these numbers as guaranteed improvements.

![WAL Insert Lock Benchmark)](/images/posts/postgresql-major-upgrade-pg13-to-pg18-lessons-learned/wal_benchmark.png)

---

## Logical Replication Improvements

Logical replication in PostgreSQL 13 was functional but limited in ways that required workarounds at scale. The versions from 14 to 17 closed most of those gaps.

**Streaming large transactions (PG14).** In PG13, any transaction too large to fit in `logical_decoding_work_mem` was spilled to disk on the publisher and only sent to the subscriber after the commit. For large bulk operations, this created a significant latency spike on the subscriber side — the entire transaction arrived at once as a single burst. PG14 added streaming mode: large in-progress transactions are now streamed to the subscriber incrementally as WAL is decoded, so the subscriber can start applying changes before the transaction commits. The burst disappears.

**Row and column filters on publications (PG14/PG15).** PG14 added `WHERE` clause filters to `CREATE PUBLICATION`, so only rows matching a predicate are replicated. PG15 added column-level filters, allowing publications to replicate a subset of columns from a table. Together these make selective replication a first-class feature rather than a workaround done with views or separate tables.

**Logical replication from standbys (PG16).** Before PG16, logical subscribers had to connect directly to the primary. Under heavy replication load this added CPU and I/O pressure to the primary. PG16 allows standbys to act as logical replication sources, offloading decoding work from the primary entirely. For clusters with many logical subscribers this is a meaningful architectural relief.

**Built-in slot failover (PG17).** In PG13, protecting logical replication slots across a primary failover required the `pg_failover_slots` extension. PG17 introduced `synchronized_standby_slots`, which replicates slot state to standbys natively. After a failover the promoted standby already has the slot with the correct LSN, and subscribers reconnect without data loss or manual intervention. Removing the `pg_failover_slots` dependency simplifies the extension inventory and the upgrade surface itself.

---

## Monitoring Improvements

Visibility into what PostgreSQL is doing internally was one of the weaker areas in PG13. Three views added between PG14 and PG16 made a concrete difference to how we diagnose problems.

**`pg_stat_wal` (PG14).** This view exposes WAL write statistics: records written, bytes written, full-page writes, and sync counts along with their total durations. Before PG14, the only way to measure WAL write overhead was through OS-level tools or indirect inference from `pg_stat_bgwriter`. `pg_stat_wal` makes WAL pressure directly visible and queryable:

```sql
SELECT wal_records, wal_bytes, wal_sync, wal_sync_time
FROM pg_stat_wal;
```

**`pg_stat_io` (PG16).** This is the most significant observability improvement in the range. `pg_stat_io` breaks down I/O operations by backend type (client backends, autovacuum workers, WAL writer, checkpointer, etc.), I/O object (relation, temp relation, WAL), and I/O context (normal, vacuum, bulkread). For the first time it is possible to answer "which process type is generating most of the read I/O right now?" directly from SQL without reaching for `iostat` or storage-level metrics:

```sql
SELECT backend_type, object, context,
       reads, writes, hits,
       read_time, write_time
FROM pg_stat_io
ORDER BY reads DESC;
```

This view became indispensable for diagnosing I/O contention between autovacuum and application traffic post-upgrade.

**`pg_stat_subscription_stats` (PG16).** For clusters with logical subscribers, this view adds per-subscription error counters: `apply_error_count` and `sync_error_count`. In PG13 a subscription that hit an error logged it and stalled, with no aggregate counter to alert on. PG16 makes subscription health alertable with a simple threshold query:

```sql
SELECT subname, apply_error_count, sync_error_count
FROM pg_stat_subscription_stats
WHERE apply_error_count > 0 OR sync_error_count > 0;
```

---

## What We Considered Before Upgrading

Deciding to upgrade is the easy part. Deciding how to upgrade safely at scale requires thinking through every dependency that `pg_upgrade` does not handle for you. Here is what we mapped out before writing a single line of automation.

**Replication topology.** Our clusters run a primary plus multiple streaming secondaries. `pg_upgrade` runs only on the primary. The secondaries must be re-synchronized afterward via rsync. This means upgrade downtime is roughly: stop services + pg_upgrade duration + rsync duration × secondary count.

**Logical replication slots are not preserved.** `pg_upgrade` does not carry over `pg_replication_slots`. If your cluster has downstream logical subscribers, their slots vanish after upgrade. The subscribers' `pg_subscription` records still exist, but there is no corresponding slot on the publisher anymore. This requires a coordinated dance: disable subscriptions on subscribers before upgrade, recreate slots on the publisher after upgrade, then re-enable and refresh subscriptions. This is one of the less-documented failure modes of production upgrades.

**Physical replication slot names.** If secondaries connect via named physical replication slots rather than `primary_slot_name` GUC, those slot names must be captured before the upgrade and recreated after starting the new PostgreSQL instance on the primary.

**Extensions.** Not all extensions ship in the standard `contrib` package. Extensions like `pg_failover_slots` or `pg_wait_sampling` require separate packages for the target version. If the package does not exist on the host before upgrade, `pg_upgrade --check` will fail. Preflight must detect all non-contrib extensions installed on the cluster.

**Backup tooling.** If Barman is managing backups for the cluster, its configuration references the source PostgreSQL binary path (`path_prefix`). After upgrade, this must be updated to point to the new binary directory. Barman also needs a fresh base backup from the upgraded cluster.

**Connection management.** Before stopping PostgreSQL, all write roles should be set to `NOLOGIN` to drain connections cleanly. Active connections should be terminated. Exporters and monitoring agents that hold connections must be stopped. All of this must happen in the right order to avoid a dirty shutdown.

**Stats and optimizer hygiene.** After upgrade, `pg_statistic` is carried over but not re-analyzed. Run `vacuumdb --analyze-in-stages` immediately after startup to rebuild statistics incrementally without flooding the system.

---

## Choosing the Target Version: PostgreSQL 17 or 18

Before writing a single line of the upgrade playbook, we had to settle a more fundamental question: upgrade to PostgreSQL 17, or go all the way to 18?

PostgreSQL 18 was already released with 18.1 available as a minor version. The case for going directly to 18 was compelling:

- **Statistics preservation.** `pg_upgrade` in PG18 preserves `pg_statistic` during the upgrade, eliminating the planner's cold-start period where query plans regress until `ANALYZE` catches up. This reduces post-upgrade risk and shortens the observation window after cutover.
- **Mature logical replication.** Slot handling, subscriber state tracking, and failover behavior are all more robust by PG18. Combined with the built-in `synchronized_standby_slots` from PG17, the `pg_failover_slots` extension dependency is gone entirely.
- **UUIDV7 support.** Native UUIDV7 generation is available without extensions. For applications that rely on time-ordered UUIDs this removes a common extension dependency.
- **B-tree Skip Scan.** The planner can now use a multi-column index even when the leading column is absent from the `WHERE` clause. Particularly effective on low-cardinality leading columns, and requires no schema changes to benefit from.

The one concern with PG18 is its asynchronous I/O subsystem. Async I/O is a brand-new code path, and at our scale we were not willing to bet a production upgrade on an I/O method with no production mileage yet. PostgreSQL 18 provides an explicit escape hatch: setting `io_method = 'sync'` in `postgresql.conf` reverts the instance to the traditional synchronous I/O path used by every prior version. With that set from day one, the async I/O risk disappears entirely while all other PG18 improvements remain.

We decided to go directly to PostgreSQL 18 with `io_method = 'sync'` and plan to revisit async I/O once it has accumulated more production mileage across the community.

---

## The Upgrade Playbook Design

We encoded all of the above into an Ansible playbook that manages the full upgrade from a single control node. The playbook runs against `localhost` and delegates all database and file operations to remote hosts via `delegate_to`. It is organized into seven sequential phases, each controlled by Ansible tags so individual phases can be re-run in isolation:

![PostgreSQL Upgrade Playbook — 7 Phases](/images/posts/postgresql-major-upgrade-pg13-to-pg18-lessons-learned/upgrade-phases.png)

**Setup** always runs and sets all facts: source/target versions, data directories, binary paths, host lists.

**Preflight** validates everything before any changes are made. This is the most important phase.

**Preparation** installs the target PostgreSQL packages, runs `initdb`, and executes `pg_upgrade --check --jobs=4` to catch any catalog-level problems before touching production.

**Upgrade** stops the cluster, runs `pg_upgrade --link --jobs=4` on the primary (using hard links so the upgrade is faster and rollback is structurally possible), and restores configuration files from pre-upgrade backups.

**Rsync** concurrently synchronizes the upgraded data from the primary to all secondaries in parallel. Rsync flags exclude log directories to reduce transfer size: `--archive --delete --hard-links --size-only --no-inc-recursive --exclude='log'`.

**Start Services** fixes ownership, handles WAL directory symlinks if a separate `/logs` mount exists, creates physical replication slots, and starts all instances.

**Post Upgrade** verifies replica connectivity, recreates logical replication slots, re-enables subscriptions, restores login on write roles, runs `vacuumdb`, updates extensions, and updates Barman configuration.

### Reducing Upgrade Duration

Downtime is the main constraint on a production upgrade. Two optimizations in the playbook made a measurable difference.

**Parallel pg_upgrade with `--jobs`.** `pg_upgrade` can parallelize the schema dump and restore pass across multiple worker processes. On clusters with a single database this matters less, but some of our clusters have 6–7 databases — without parallelism, each database is processed sequentially. Setting `--jobs=4` keeps all four workers busy across databases and cuts the pg_upgrade runtime significantly. Both the dry-run (`--check`) and the live upgrade use the same flag:

```yaml
- name: "[primary] Execute pg_upgrade on primary"
  ansible.builtin.command:
    cmd: >
      {{ target_pg_bin_dir }}/pg_upgrade
      --old-datadir={{ source_pg_data_dir }}
      --new-datadir={{ target_pg_data_dir }}
      --old-bindir={{ source_pg_bin_dir }}
      --new-bindir={{ target_pg_bin_dir }}
      --link
      --jobs=4
  delegate_to: "{{ primary_instance }}"
  become_user: postgres
  register: pg_upgrade_result
  timeout: "{{ pg_upgrade_timeout }}"
```

**Async rsync to all secondaries simultaneously.** Without async execution, rsync would run against each secondary in sequence — downtime would grow linearly with replica count. The playbook fires all rsync jobs in parallel using Ansible's `async` / `poll: 0` pattern, then waits for all of them together. The effective rsync duration becomes that of the slowest secondary rather than the sum of all of them:

```yaml
- name: "[primary] Rsync data directories from primary to secondary instances"
  ansible.builtin.shell:
    cmd: >-
      rsync --archive --delete --hard-links --size-only
      --no-inc-recursive --exclude='log' --exclude='pg_wal_old'
      -e "ssh -i /tmp/pg_rsync_temp_key -o StrictHostKeyChecking=no"
      {{ source_pg_rsync_dir }} {{ target_pg_rsync_dir }}
      root@{{ item }}:/var/lib/pgsql
  delegate_to: "{{ primary_instance }}"
  loop: "{{ secondary_instances }}"
  register: rsync_results
  async: "{{ rsync_timeout }}"
  poll: 0
```

**Post-rsync file count validation.** After all rsync jobs finish, the playbook counts files under `base/` on the primary and each secondary and asserts the numbers match. A mismatch means a partial transfer — starting PostgreSQL on top of it would produce a corrupt replica. The assertion hard-fails the playbook before any instance is started:

```yaml
- name: "[ansible] Fail if base directory file count on secondary does not match primary"
  ansible.builtin.assert:
    that:
      - item.stdout | trim == primary_base_file_count.stdout | trim
    fail_msg: >-
      File count mismatch: primary has {{ primary_base_file_count.stdout | trim }} files
      but {{ item.item }} has {{ item.stdout | trim }} files in {{ target_pg_data_dir }}/base
  loop: "{{ secondary_base_file_counts.results }}"
  when: secondary_instances | length > 0
```

---

## Preflight: The Most Important Phase

The preflight phase is designed to surface every problem before the cluster is touched. Some of the checks that took significant iteration to get right:

- **Inactive replication slot detection.** A physical slot that has not been consumed will hold back WAL deletion and can cause the upgrade to carry over stale slot state. Preflight alerts the operator before proceeding.
- **Redo LSN sync check.** On a primary + secondaries cluster, if the secondaries are not fully caught up, rsyncing the upgraded primary data on top of a diverged secondary will produce a broken cluster. Preflight compares `pg_control_checkpoint` redo LSN across all instances.
- **Extension inventory.** Preflight computes which installed extensions are not in the standard `contrib` list and surfaces them explicitly. The operator must confirm those packages are available for the target version before proceeding.
- **WAL directory size.** We added a check that resolves the effective WAL path (including symlink detection), measures its size in GB, warns if it exceeds 500 GB, and flags any stale `pg_wal_*` directories left from previous upgrade attempts.
- **Logical subscriber detection.** Preflight queries `pg_stat_replication` and `pg_replication_slots` to identify downstream logical subscribers, generates the slot recreation SQL, and verifies SSH connectivity to those subscriber hosts.

The preflight phase ends with a human-readable summary and an explicit operator confirmation prompt before any destructive phase begins.

---

## Why Upgrade Process Was Dangerous and Risky Still

### Logical Replication: Two Scenarios

The logical replication handling required the most careful design because there are two completely different scenarios depending on the role of the instance being upgraded. The playbook handles both, but the sequence and the SQL differ in a way that matters.

**Scenario A — This instance is a publisher.** Other PostgreSQL instances subscribe to this cluster. Their replication slots (`pg_replication_slots`) do not survive `pg_upgrade`. The playbook handles this across three phases:

During **preflight**, the playbook discovers all downstream subscribers by querying `pg_replication_slots` and `pg_stat_replication`, collects each subscriber's IP, subscription name, slot name, and database, and verifies SSH connectivity to each subscriber host. It also pre-generates the slot recreation SQL and saves it to `/tmp/logical_slot.sql` on the primary — before any upgrade step runs.

During the **upgrade phase**, the playbook connects to each subscriber directly and disables its subscription:

```sql
ALTER SUBSCRIPTION your_subscription DISABLE;
ALTER SUBSCRIPTION your_subscription SET (slot_name = NONE);
```

It then asserts that each subscription is confirmed disabled (`subenabled = f`) and slot detached (`subslotname = NULL`) before proceeding with `pg_upgrade`.

During **post-upgrade**, the playbook recreates the logical slots by executing `/tmp/logical_slot.sql` on the upgraded primary, then re-attaches and re-enables each subscription on the subscriber:

```sql
ALTER SUBSCRIPTION your_subscription SET (slot_name = 'your_slot');
ALTER SUBSCRIPTION your_subscription ENABLE;
ALTER SUBSCRIPTION your_subscription REFRESH PUBLICATION;
```

Note that `REFRESH PUBLICATION` is called here — without `copy_data = false` — as a safety step to pick up any table membership changes that occurred while the subscription was disabled. Because the subscriber itself was not upgraded, `pg_subscription_rel` is intact and no data is re-copied. Finally, the playbook queries `pg_replication_slots` on the primary and asserts that no logical slots remain inactive, confirming every subscriber has reconnected successfully.

**Scenario B — This instance is a subscriber.** The subscription definitions survive in `pg_subscription`, but `pg_subscription_rel` (which tracks per-table replication state) is not preserved by `pg_upgrade`. After upgrade the subscriber has no record of which tables to replicate. The playbook disables all own subscriptions before the upgrade identically to Scenario A, then after upgrade re-enables them with:

```sql
ALTER SUBSCRIPTION your_subscription SET (slot_name = 'your_slot');
ALTER SUBSCRIPTION your_subscription ENABLE;
ALTER SUBSCRIPTION your_subscription REFRESH PUBLICATION WITH (copy_data = false);
```

The `copy_data = false` flag is critical here. Without it, PostgreSQL treats the refresh as an initial sync and re-copies the full table data from the publisher. With it, `pg_subscription_rel` is repopulated using the existing replication origin state, and replication resumes from where it left off.

The critical difference between the two scenarios is not whether `REFRESH PUBLICATION` runs — it runs in both — but whether `copy_data = false` is included. Getting this wrong in the subscriber direction triggers an accidental full table copy. Getting it wrong in the publisher direction (adding `copy_data = false` when `pg_subscription_rel` is already gone) causes a silent replication stall.

### The Constraint That Existed Twice

This is the first real bug we discovered — one that caused `pg_upgrade` to fail partway through the `pg_restore` phase, and which we raised with the PostgreSQL community.

#### The Root Cause

In PostgreSQL 13 and earlier, `NOT NULL` column constraints are not stored as rows in `pg_constraint`. They are stored as a boolean flag `attnotnull` in `pg_attribute`. In PostgreSQL 17 and later, `NOT NULL` constraints are stored as first-class rows in `pg_constraint` with `contype = 'n'`.

During `pg_upgrade`, when migrating the schema from PG13 to PG17+, PostgreSQL needs to create these new `NOT NULL` constraint rows. It generates constraint names automatically using the pattern `{table_name}_{column_name}_not_null`. It then inserts them into `pg_catalog.pg_constraint`, which has a unique index:

```text
pg_constraint_conrelid_contypid_conname_index UNIQUE (conrelid, contypid, conname)
```

Now consider a common pattern in production schemas: a `CHECK` constraint that explicitly validates `NOT NULL`, often used before PostgreSQL had proper `NOT NULL` constraint support, or carried over from ORMs and migration tools. If that check constraint happens to be named `{table_name}_{column_name}_not_null`, you have a collision. `pg_upgrade` tries to insert a new row with the same name that already exists in the constraint catalog — and the unique index rejects it.

#### The Scenario That Triggers It

```sql
CREATE TABLE orders (
    id BIGSERIAL,
    customer_id INTEGER NOT NULL,
    CONSTRAINT orders_customer_id_not_null CHECK (customer_id IS NOT NULL)
) PARTITION BY RANGE (order_date);
```

Before the upgrade (PG13), `pg_constraint` only shows the `CHECK` constraint:

```text
conname                      | table_name | contype
orders_customer_id_not_null  | orders     | c
orders_customer_id_not_null  | orders_2024_01 | c
orders_customer_id_not_null  | orders_2024_02 | c
```

After the upgrade (PG17+), PostgreSQL tries to add:

```text
orders_customer_id_not_null  | orders     | n   ← NEW not_null constraint
orders_2024_01_customer_id_not_null | orders_2024_01 | n
orders_2024_02_customer_id_not_null | orders_2024_02 | n
```

For the parent table `orders`, the new `not_null` constraint name collides with the existing check constraint name. `pg_restore` fails with a duplicate key violation and the entire upgrade aborts. For partitioned tables this is compounded because every partition inherits the check constraint, multiplying the number of collisions.

#### The Fix

The workaround is to rename the conflicting check constraints before the upgrade. Use the following query to identify all candidates:

```sql
SELECT
    conrelid::regclass AS table_name,
    conname AS constraint_name,
    substring(pg_get_constraintdef(oid) FROM 'CHECK \(\((\w+) IS NOT NULL\)\)') AS column_name,
    'ALTER TABLE ' || conrelid::regclass::text
        || ' RENAME CONSTRAINT ' || conname
        || ' TO ' || conrelid::regclass::text
        || '_' || substring(pg_get_constraintdef(oid) FROM 'CHECK \(\((\w+) IS NOT NULL\)\)')
        || '_not_null1;' AS rename_script
FROM pg_constraint
WHERE contype = 'c'
  AND conname LIKE '%\_not\_null' ESCAPE '\'
  AND pg_get_constraintdef(oid) LIKE '%NOT NULL%';
```

This query surfaces every `CHECK (column IS NOT NULL)` constraint whose name matches the pattern that `pg_upgrade` will attempt to generate. The `rename_script` column produces the `ALTER TABLE ... RENAME CONSTRAINT` statements you need to run.

One important caveat: **you cannot rename a constraint on an inherited child partition directly**. The rename must happen on the parent table. PostgreSQL will propagate it to the children.

#### Community Bug Report

After hitting this failure in production, we filed a bug report with the PostgreSQL community mailing list. We provided a minimal reproduction case, traced the code path through `pg_upgrade`'s restore phase, and reviewed the proposed patch. The bug was confirmed by the community and the fix was merged. **This bug is resolved in PostgreSQL 18.2.** You can follow the full discussion and patch review here: [postgresql.org mailing list thread](https://www.postgresql.org/message-id/19393-6a82427485a744cf%40postgresql.org).

After the upgrade succeeds, the renamed check constraints become redundant because PG17+ now tracks `NOT NULL` natively. Use this query on the upgraded cluster to generate the cleanup:

```sql
SELECT 'ALTER TABLE ' || c.conrelid::regclass || ' DROP CONSTRAINT ' || c.conname || ';'
FROM pg_constraint c
WHERE c.contype = 'c'
  AND pg_get_constraintdef(c.oid) LIKE '%IS NOT NULL%'
  AND c.conparentid = 0;
```

Only drop parent constraints — child partition constraints are dropped automatically.

#### Summary for This Problem

1. Before upgrading, run the detection query on every database in the cluster.
2. If rows are returned, rename the matching constraints using the generated scripts.
3. Test `pg_upgrade --check` on a clone before touching production.
4. After upgrading, drop the now-redundant check constraints.

### The Role That Was Gone But Not Forgotten

The second problem is subtler and harder to anticipate. It surfaces only under a specific sequence of operations that is not uncommon in long-running production environments: a database was renamed, its owner role was reassigned and dropped, and later the cluster was upgraded.

#### Background: What is pg_init_privs

`pg_init_privs` is a system catalog that records the initial privileges of objects at the time they were created — either by `initdb` or by `CREATE EXTENSION`. When you install an extension, PostgreSQL records which roles were granted privileges on the extension's objects (functions, views, tables) at install time. These records serve as the baseline for privilege comparison and are used during `pg_upgrade` to restore the original access control state on the new cluster.

The important thing to understand is that `pg_init_privs` records role OIDs, not role names. If a role is later dropped, the OID reference in `pg_init_privs` becomes a dangling pointer. There is no foreign key constraint enforcing referential integrity here.

#### The Sequence That Produces Orphan OIDs

```sql
-- 1. Create a database with a dedicated owner role
CREATE DATABASE my_db OWNER benchmark_owner;
\c my_db
SET ROLE benchmark_owner;

-- 2. Install an extension that writes to pg_init_privs
CREATE EXTENSION pg_wait_sampling;
RESET ROLE;
```

At this point `pg_init_privs` contains rows like:

```text
objoid | classoid | privtype | initprivs
-------+----------+----------+------------------------------------------
16425  | 1259     | e        | {16461=arwdDxt/16461,=r/16461}
16430  | 1259     | e        | {16461=arwdDxt/16461,=r/16461}
16435  | 1259     | e        | {16461=arwdDxt/16461,=r/16461}
16439  | 1255     | e        | {16461=X/16461}
```

OID `16461` is the OID of `benchmark_owner`. Now, some time later, the database is renamed, the role is reassigned, and the role is dropped:

```sql
-- Terminate connections first
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity WHERE datname = 'my_db';

ALTER DATABASE my_db RENAME TO my_db_v2;
\c my_db_v2

REASSIGN OWNED BY benchmark_owner TO postgres;
DROP OWNED BY benchmark_owner;
\c postgres
DROP ROLE benchmark_owner;
```

After this sequence, `pg_init_privs` still contains rows referencing OID `16461`. PostgreSQL does not clean these up when a role is dropped. Verify this with:

```sql
SELECT pip.objoid
FROM pg_init_privs pip
CROSS JOIN LATERAL aclexplode(pip.initprivs) ace
LEFT JOIN pg_authid a ON a.oid = ace.grantee
WHERE a.oid IS NULL
AND ace.grantee <> 0;
```

If this query returns rows, you have dangling OID references in `pg_init_privs`.

#### How This Breaks pg_upgrade

During `pg_upgrade`, the schema of each database is dumped with `pg_dump` and then restored with `pg_restore` into the new cluster. When `pg_restore` processes the access control entries for `pg_wait_sampling`'s functions, it encounters the `pg_init_privs` record and generates SQL like:

```sql
SELECT pg_catalog.binary_upgrade_set_record_init_privs(true);
REVOKE ALL ON FUNCTION "public"."pg_wait_sampling_reset_profile"() FROM PUBLIC;
REVOKE ALL ON FUNCTION "public"."pg_wait_sampling_reset_profile"() FROM "postgres";
SET SESSION AUTHORIZATION "16461";
GRANT ALL ON FUNCTION "public"."pg_wait_sampling_reset_profile"() TO "16461";
RESET SESSION AUTHORIZATION;
SELECT pg_catalog.binary_upgrade_set_record_init_privs(false);
```

The OID `16461` has been serialized as a literal role identifier in the restore script. When `pg_restore` executes `SET SESSION AUTHORIZATION "16461"`, PostgreSQL looks up a role named `"16461"` — and finds nothing, because that role no longer exists. The error:

```text
pg_restore: error: could not execute query: ERROR: role "16461" does not exist
```

The restore fails with `--exit-on-error` and `pg_upgrade` aborts.

#### The Fix: Clean Up Orphan OIDs Before Upgrading

The mitigation is to run preflight checks against `pg_init_privs` across all databases before starting `pg_upgrade`. There are three categories to check:

**Check 1 — Dangling role OIDs in initprivs:**

```sql
SELECT pip.objoid, pip.initprivs, e.extname
FROM pg_init_privs pip
CROSS JOIN LATERAL aclexplode(pip.initprivs) ace
LEFT JOIN pg_authid a ON a.oid = ace.grantee
JOIN pg_depend d ON d.objid = pip.objoid
JOIN pg_extension e ON e.oid = d.refobjid
WHERE a.oid IS NULL
AND ace.grantee <> 0;
```

**Check 2 — Orphaned pg_init_privs rows where the object itself no longer exists:**

```sql
SELECT pip.*
FROM pg_init_privs pip
WHERE NOT EXISTS (SELECT 1 FROM pg_class c WHERE c.oid = pip.objoid)
AND NOT EXISTS (SELECT 1 FROM pg_proc p WHERE p.oid = pip.objoid)
AND NOT EXISTS (SELECT 1 FROM pg_type t WHERE t.oid = pip.objoid);
```

**Check 3 — Extensions whose owner role no longer exists:**

```sql
SELECT e.extname, e.extowner, a.rolname
FROM pg_extension e
LEFT JOIN pg_authid a ON a.oid = e.extowner
WHERE a.rolname IS NULL;
```

After identifying the orphaned records, clean them up:

```sql
-- Remove dangling role OIDs from initprivs
DELETE FROM pg_init_privs
WHERE objoid IN (
    SELECT pip.objoid
    FROM pg_init_privs pip
    CROSS JOIN LATERAL aclexplode(pip.initprivs) ace
    LEFT JOIN pg_authid a ON a.oid = ace.grantee
    WHERE a.oid IS NULL
    AND ace.grantee <> 0
);

-- Remove rows whose objects no longer exist
DELETE FROM pg_init_privs
WHERE NOT EXISTS (SELECT 1 FROM pg_class c WHERE c.oid = objoid)
AND NOT EXISTS (SELECT 1 FROM pg_proc p WHERE p.oid = objoid)
AND NOT EXISTS (SELECT 1 FROM pg_type t WHERE t.oid = objoid);
```

Run these checks and cleanups on every database in the cluster before running `pg_upgrade --check`. Any database that returns rows from Check 1 or Check 2 is a potential upgrade failure.

We added this check as a mandatory preflight step in the playbook, tightened to filter by `privtype = 'e'` (extension-granted privileges) and extended to cover `pg_namespace` objects as well.

---

## Post-Upgrade Metrics and Validation

Completing the upgrade is not the finish line — it is the start of the measurement window. The areas below are the key signals to track after cutover to confirm the upgrade delivered what you expected and to catch any regressions early.

### Log Prerequisites

Before the upgrade, ensure these settings are active so baseline data is captured in the logs for comparison:

```
log_destination         = 'csvlog'
logging_collector       = on
log_checkpoints         = on
log_autovacuum_min_duration = 250ms
log_min_duration_statement  = 1000
```

Without `log_checkpoints` and `log_autovacuum_min_duration` set before the upgrade, there is no pre-upgrade baseline to compare against.

### Error and Fatal Rate

The first thing to check is the `ERROR`, `FATAL`, and `PANIC` rate. The total error count should not increase after the upgrade. A spike — especially new `FATAL` lines — indicates something wrong with cluster configuration or extension compatibility and needs investigation before anything else.

Constraint-level errors (duplicate key violations, constraint violations, replication conflicts) should be stable across the upgrade since they are application-driven, not infrastructure-driven.

### Slow Query Volume

After the upgrade, with faster autovacuum and reduced I/O contention, the count of slow queries logged by `log_min_duration_statement` should drop. A significant reduction here is the most direct confirmation that the performance improvements translated into real workload gains on your cluster.

### Cancelled Queries

Lock timeouts and statement timeouts are counted separately. Statement timeout cancellations are particularly informative: many of them are caused by queries blocked behind autovacuum I/O, which competes far less aggressively in PG17+. A reduction in `statement_timeout` cancellations after the upgrade — without any application-side changes — is a strong signal that autovacuum is no longer a hidden source of query latency.

Lock timeout counts should remain stable. An increase there is worth investigating as a potential regression in locking behavior.

### Checkpoint Duration

Track average `write`, `sync`, and `total` checkpoint times from `log_checkpoints` output before and after. Checkpoint improvements in PG18 come primarily from better I/O scheduling and the async I/O path (which we disabled with `io_method = sync`, so this category should show modest improvement at best). What to watch for: the `sync` time. A large sync time before the upgrade suggests I/O pressure during checkpoints that the new version may alleviate through better `maintenance_io_concurrency` handling even without async I/O.

### Autovacuum Duration

This is the most expected win. Compare autovacuum run count, average duration per run, and total time spent in autovacuum before and after using `log_autovacuum_min_duration` output. On our clusters the average duration per run dropped by over 70%, and the total accumulated autovacuum time across the observation window dropped by a similar margin. Fewer and shorter autovacuum runs mean less I/O competition with application queries — which feeds back into the slow query and timeout counts above.

---

## Conclusion: A Checklist for DBAs

A PostgreSQL major version upgrade is manageable if you do the right preparation. Here is the condensed checklist from everything above:

**Before running pg_upgrade --check:**

- [ ] Inventory all installed extensions. Verify the target version package exists for every non-contrib extension on every host.
- [ ] Detect inactive physical or logical replication slots. Resolve or drop them.
- [ ] Check all databases for `CHECK (column IS NOT NULL)` constraints whose name matches `{table}_{column}_not_null`. Rename any that match.
- [ ] On every database, run the `pg_init_privs` orphan OID checks. Clean up any dangling role references.
- [ ] Verify `pg_upgrade --check` passes cleanly on a production clone before the maintenance window.
- [ ] Confirm secondaries are fully caught up (redo LSN matches primary).
- [ ] Measure WAL directory size. Warn if it exceeds 500 GB.

**During the upgrade window:**

- [ ] Set write roles to `NOLOGIN` before stopping PostgreSQL.
- [ ] Disable logical subscriptions on downstream subscribers before stopping the publisher.
- [ ] Generate slot recreation SQL before running `pg_upgrade`.
- [ ] Use `pg_upgrade --link` to avoid full data copy (but note: source data dir becomes unusable if the new cluster writes).
- [ ] Rsync data to secondaries concurrently to minimize downtime.

**After the upgrade:**

- [ ] Recreate logical replication slots and re-enable subscriptions.
- [ ] Run `REFRESH PUBLICATION WITH (copy_data = false)` on upgraded subscribers.
- [ ] Run `vacuumdb --analyze-in-stages` to rebuild optimizer statistics.
- [ ] Update extensions that require schema changes (e.g., `ALTER EXTENSION pg_stat_statements UPDATE`).
- [ ] Drop redundant `CHECK (IS NOT NULL)` constraints that are now covered by native `NOT NULL` constraints.
- [ ] Update Barman `path_prefix` to point to the new binary directory and take a fresh base backup.

---

Upgrading PostgreSQL across a major version boundary is never just a package swap. The gains from PG13 to PG18 — faster autovacuum, better WAL scalability, async I/O — are real and measurable, but so are the failure modes. The two bugs we hit were not edge cases in theory; they were invisible until the upgrade ran. Finding them cost us time, but filing the bug report and watching the patch land in PG18.2 made it worthwhile.

The checklist above is everything we wish we had before we started. Run it on a clone first, fix what it surfaces, then run the real upgrade with confidence.

This post covers the playbook and the catalog-level bugs. It does not cover what happens inside a subscriber's replication origin during the upgrade window — that failure mode is subtle enough to deserve its own post: [Upgrading a PostgreSQL Subscriber: Why pg_upgrade Can Silently Replay WAL You Already Applied](/posts/upgrading-postgresql-logical-replication-subscribers-with-pg-upgrade/).

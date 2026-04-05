+++
title = 'Same Cluster, Different Rules. synchronous_standby_names vs synchronized_standby_slots'
date = '2026-04-05'
draft = false
description = 'Understanding why synchronous_standby_names and synchronized_standby_slots are independent in PostgreSQL, and how misconfiguring them together can silently break your HA setup'
tags = ['postgresql', 'replication', 'high-availability', 'configuration', 'logical-replication', 'database']
author = 'kylorend3r'
+++

## Introduction

PostgreSQL gives you powerful tools to build highly available setups with synchronous replication and logical slot failover. But the confusion comes from that`synchronous_standby_names` and `synchronized_standby_slots` look related, sound related, and live in the same configuration file. Yet they govern completely different things, and assuming otherwise can silently break your setup in ways that are hard to diagnose under pressure.

`synchronous_standby_names` is about transaction durability. It controls which standbys must confirm receipt of WAL before a commit is acknowledged back to the client. `synchronized_standby_slots`, introduced in PostgreSQL 17, is about something else entirely. It controls which physical replication slots must confirm WAL receipt before the primary advances its logical walsenders. In other words, one governs when your application gets a commit acknowledgment, and the other governs when your logical replication slots are considered safe to advance. They operate on separate code paths, follow separate rules, and one does not imply the other.

The problem is that when you put both of these parameters in play, as you would in a modern HA setup combining synchronous replication with logical replication slot failover, the overlap in naming and the similarity of their syntax creates a strong intuition that they should be configured symmetrically. That intuition is wrong, and the consequences range from unexpected write latency spikes to replication stalls when a standby falls behind or goes down.

Let me illustrate this. Say you want to use logical replication slots with failover enabled, and at the same time you have synchronous standby configured in the cluster.

```
benchmark=# SELECT pg_create_logical_replication_slot(
  'cdc_slot_for_debezium',
  'pgoutput',
  false,     -- temporary
  true       -- failover = true  ← this is the key flag
);
 pg_create_logical_replication_slot
------------------------------------
 (cdc_slot_for_debezium,A/E8137588)
(1 row)

benchmark=# select * from pg_replication_slots where slot_name='cdc_slot_for_debezium';
       slot_name       |  plugin  | slot_type | datoid | database  | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size | two_phase |        inactive_since        | conflicting | invalidation_reason | failover | synced
-----------------------+----------+-----------+--------+-----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------+-----------+------------------------------+-------------+---------------------+----------+--------
 cdc_slot_for_debezium | pgoutput | logical   |  16463 | benchmark | f         | f      |            |      |      1419897 | A/E8137550  | A/E8137588          | reserved   |               | t         | 2026-04-05 13:16:02.66053+00 | f           |                     | f        | f
(1 row)
```

In this post, we'll walk through how each parameter actually works under the hood, why they are independent by design, and what combinations of misconfiguration look like in practice. That way you can reason about your topology with confidence before something goes sideways at 2 AM.

![Topology diagram showing the independence of synchronous_standby_names and synchronized_standby_slots](/images/posts/synchronous-standby-names-vs-synchronized-standby-slots/topology.webp)

*The solid blue arrows show the WAL stream path governed by `synchronous_standby_names` (commit ACK). The dashed green arrows show the slot sync path governed by `synchronized_standby_slots` (logical walsender advancement). Same standbys, entirely separate control planes.*

## Table of Contents

- [Introduction](#introduction)
- [Table of Contents](#table-of-contents)
- [Background: synchronous\_standby\_names](#background-synchronous_standby_names)
- [Background: synchronized\_standby\_slots](#background-synchronized_standby_slots)
- [Why They Are Independent](#why-they-are-independent)
- [Misconfiguration Scenarios](#misconfiguration-scenarios)
  - [Scenario 1: Adding a non-failover candidate to synchronized\_standby\_slots](#scenario-1-adding-a-non-failover-candidate-to-synchronized_standby_slots)
  - [Scenario 2: Assuming synchronous\_standby\_names covers logical slots](#scenario-2-assuming-synchronous_standby_names-covers-logical-slots)
  - [Scenario 3: ANY quorum in synchronous\_standby\_names creates false confidence](#scenario-3-any-quorum-in-synchronous_standby_names-creates-false-confidence)
- [Correct Configuration Example](#correct-configuration-example)
- [Conclusion](#conclusion)
- [References](#references)

## Background: synchronous_standby_names

Synchronous replication in PostgreSQL is a commit-level guarantee. When `synchronous_standby_names` is set, the primary will not return a success response to the client until the named standbys have confirmed that they received and wrote the WAL for that transaction. The client waits. The application waits. Nothing proceeds until the standby confirms.

```bash
# primary postgresql.conf
synchronous_standby_names = '1(standby_1, standby_2)'
```

The syntax lets you be specific: `1(standby_1, standby_2)` means at least one of those two must confirm. `2(standby_1, standby_2)` means both must. You can also use `FIRST` and `ANY` keywords for more nuanced quorum configurations. The standbys are identified by their `application_name` in `primary_conninfo`.

The key thing to internalize is what "confirmation" means here. By default (`synchronous_commit = on`), the standby must confirm that WAL has been written to its own disk. With `synchronous_commit = remote_apply`, it must go further and confirm the WAL has been applied. The level of guarantee is configurable, but in all cases it is tied directly to the transaction commit path on the primary.

The trade-off is latency. Every commit that falls under `synchronous_standby_names` incurs at least one network round-trip to the standby. If that standby is slow, lagging, or unreachable, your writes stall. This is the intended behavior: you are trading latency for durability. It is important to understand that this parameter reaches directly into your application's write path.

## Background: synchronized_standby_slots

Starting with PostgreSQL 17, it became possible to sync logical replication slots from the primary to standbys automatically, so that after a failover the new primary can resume logical replication from where the old one left off. This is a significant operational improvement. Previously, logical slots were lost on failover and had to be recreated manually.

The feature relies on a few moving parts working together. On the standby, `sync_replication_slots = on` activates a background worker called the slotsync worker, which periodically copies the state of logical slots marked with `failover = true` from the primary. The standby also needs a physical replication slot configured via `primary_slot_name`, and `hot_standby_feedback = on` must be enabled. The physical slot is not optional. The slotsync worker uses it to retain WAL and catalog xmin information on the primary so that the logical slots being copied can actually catch up.

On the primary side, `synchronized_standby_slots` tells PostgreSQL which physical slots (corresponding to standbys running the slotsync worker) must have confirmed receiving WAL up to a certain point before the primary's logical walsenders are allowed to advance.

```bash
# primary postgresql.conf
synchronized_standby_slots = 'standby_physical_slot'
```

The intent is a safety guarantee: before the primary tells a logical replication subscriber "you're caught up to LSN X," it ensures the standbys named in `synchronized_standby_slots` have also received WAL up to that point. This way, if the primary dies and a standby is promoted, the logical slot on the new primary is guaranteed to be at least as far ahead as the subscriber's last confirmed position, so replication can resume without data loss or gaps.

To create a logical slot that participates in this failover mechanism, the `failover` flag must be set explicitly:

```sql
-- primary: create a failover-enabled logical slot
SELECT pg_create_logical_replication_slot(
  'my_logical_slot',
  'pgoutput',
  false,   -- not temporary
  true     -- failover = true
);
```

## Why They Are Independent

Now that both parameters are clear individually, the independence becomes straightforward to explain, but it is worth being explicit about.

`synchronous_standby_names` operates on the **commit path**. It intercepts every transaction commit on the primary and holds it until the named standbys reply. It does not know or care about replication slots. It does not know or care about logical replication. It is purely about WAL durability acknowledgment at the transaction level.

`synchronized_standby_slots` operates on the **logical walsender path**. It intercepts the advancement of logical replication progress on the primary and holds it until the named physical slots confirm they have received the relevant WAL. It does not affect transaction commits. It does not affect streaming replication. It is purely about keeping failover-ready logical slots in sync.

Here is a direct comparison of the two:

| | `synchronous_standby_names` | `synchronized_standby_slots` |
|---|---|---|
| **Introduced** | Long-standing | PostgreSQL 17 |
| **Set on** | Primary | Primary |
| **Identifies standbys by** | `application_name` | Physical slot name |
| **Blocks** | Transaction commits | Logical walsender advancement |
| **Purpose** | Synchronous replication durability | Logical slot failover readiness |
| **Affects write latency** | Yes, directly | Only for logical replication subscribers |
| **Requires physical slot** | No | Yes (`primary_slot_name` on standby) |

The critical thing to notice in that table: they identify standbys in completely different ways. `synchronous_standby_names` uses `application_name`. `synchronized_standby_slots` uses physical slot names. There is no shared registry, no cross-checking, and no inheritance between them. You can have the same standby appear in both, or in neither, or in only one. PostgreSQL will not warn you either way.

## Misconfiguration Scenarios

### Scenario 1: Adding a non-failover candidate to synchronized_standby_slots

Imagine a topology with three standbys: two real failover candidates and one read replica that you would never promote. You want logical slot failover to work for the two real candidates, so you run `sync_replication_slots = on` on all three for convenience. On the primary you then set:

```bash
synchronized_standby_slots = 'phys_s1, phys_s2, phys_s3'
```

What happens? The primary's logical walsenders will now wait for all three physical slots, including `phys_s3` on your read replica, to confirm WAL before advancing. If that read replica is under heavy query load, has network issues, or is intentionally lagging behind, your logical replication subscribers on the primary will stall. Not intermittently, but consistently, every time that replica falls behind.

Keep in mind that the read replica may have `sync_replication_slots = on` and be happily syncing slots, but that is a separate concern from whether it should be blocking logical replication advancement on the primary. The correct fix is straightforward:

```bash
# Only list the true failover candidates
synchronized_standby_slots = 'phys_s1, phys_s2'
```

The read replica (`phys_s3`) can still run the slotsync worker and keep its slots up to date. It just won't block the primary.

### Scenario 2: Assuming synchronous_standby_names covers logical slots

This is the more dangerous scenario because it is an invisible misconfiguration: everything appears to be working until a failover happens.

```bash
synchronous_standby_names = '1(standby_1, standby_2)'
```

Logical replication is also in use, with `failover = true` slots created on the primary. The DBA assumes that since `standby_1` and `standby_2` are already confirming every WAL write for synchronous commits, the logical slots must also be synced to those standbys. After all, the standbys have all the WAL, right?

The problem is that having the WAL is not the same as having the logical slot state synced. The `confirmed_flush` LSN, the `catalog_xmin`, the slot's restart LSN: these are tracked in the slot's own state file, and they are copied by the slotsync worker, not by streaming replication. A standby that receives all WAL via streaming replication will not automatically have a usable copy of a logical slot unless `sync_replication_slots = on` is configured and `synchronized_standby_slots` is set on the primary.

After a failover in this scenario, the promoted standby has no logical slot state. The subscriber connects, finds no matching slot, and logical replication stops. In the best case this is a brief disruption. In the worst case, if the subscriber has been advancing and the slot would need to restart from an older LSN, WAL may no longer be available.

The fix requires both sides of the configuration to be in place:

```bash
# primary postgresql.conf
synchronized_standby_slots = 'phys_standby_1'  # physical slot name on standby_1

# standby_1 postgresql.conf
primary_conninfo = 'host=primary port=5432 user=replicator dbname=postgres'
primary_slot_name = 'phys_standby_1'
hot_standby_feedback = on
sync_replication_slots = on
```

### Scenario 3: ANY quorum in synchronous_standby_names creates false confidence

This scenario is arguably the most dangerous because the system appears healthy from the application's point of view: commits succeed, replication is running, but your logical replication pipeline is silently frozen.

Consider this configuration:

```bash
# primary postgresql.conf
synchronous_standby_names = 'ANY 1(standby_1_node, standby_2_node)'
synchronized_standby_slots = 'phys_s1, phys_s2'
```

The `ANY 1` quorum means PostgreSQL only needs one of the two standbys to confirm WAL before acknowledging a commit. This is a common and reasonable choice for HA setups. It tolerates one standby going down without stalling writes. Now standby_2 goes offline.

From the commit path's perspective, `standby_1_node` alone satisfies `ANY 1`. Commits continue. The application sees no errors. Streaming replication to standby_1 is healthy.

From the logical walsender path's perspective, things are very different. `synchronized_standby_slots` has no quorum syntax: it is always an implicit ALL. Both `phys_s1` and `phys_s2` must confirm WAL before the primary advances its logical walsenders. Since `phys_s2` is on the downed standby, it stops sending confirmations. The primary's logical walsenders stall.

The result is a split state:

```text
commits:               ✓  flowing normally (ANY 1 satisfied by standby_1)
streaming replication: ✓  healthy to standby_1
logical replication:   ✗  frozen, subscriber receives no new changes
```

Your CDC pipeline, your logical replication subscriber, your downstream data warehouse: all of them stop receiving changes silently. There is no error on the primary. The subscriber connection stays open. The slot exists and looks valid in `pg_replication_slots`. The only signal is that `confirmed_flush_lsn` on the slot stops advancing.

You can observe this by comparing the slot's `confirmed_flush_lsn` against the current WAL position:

```sql
SELECT slot_name,
       confirmed_flush_lsn,
       pg_current_wal_lsn(),
       pg_current_wal_lsn() - confirmed_flush_lsn AS lag_bytes
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

If `lag_bytes` is growing steadily while the subscriber appears connected, a stalled walsender due to `synchronized_standby_slots` is a likely cause.

The fix is to align the two parameters' tolerance levels intentionally. If you are using `ANY 1` in `synchronous_standby_names` because you accept the risk of one standby being down, you should list only one standby in `synchronized_standby_slots`, specifically the one you are most confident will stay up or the designated primary failover target:

```bash
# primary postgresql.conf
synchronous_standby_names = 'ANY 1(standby_1_node, standby_2_node)'

# Only the primary failover candidate — logical replication can tolerate
# standby_2 being down, matching the same tolerance as synchronous replication.
synchronized_standby_slots = 'phys_s1'
```

If you genuinely need both standbys to have synced slots for failover safety, then accept that losing either standby will stall logical replication, and make sure your monitoring catches that before it goes unnoticed for hours.

## Correct Configuration Example

When running both synchronous replication and logical slot failover together, the key is to think about each parameter's scope separately and make deliberate choices about which standbys appear in each list.

Below is an annotated example for a primary with two failover candidates and one read replica:

```bash
# primary postgresql.conf

# Synchronous replication: require at least 1 of the 2 failover standbys
# to confirm WAL before commit is acknowledged to the client.
# Uses application_name from the standby's primary_conninfo.
synchronous_standby_names = '1(standby_1, standby_2)'

# Logical slot failover: both failover standbys must confirm WAL receipt
# before logical walsenders advance. Uses physical slot names.
# Do NOT include phys_read_replica here — it would stall logical replication.
synchronized_standby_slots = 'phys_standby_1, phys_standby_2'
```

```bash
# standby_1 postgresql.conf (failover candidate)
primary_conninfo = 'host=primary port=5432 user=replicator dbname=postgres application_name=standby_1'
primary_slot_name = 'phys_standby_1'   # physical slot — required for slotsync
hot_standby_feedback = on              # required for catalog_xmin tracking
sync_replication_slots = on            # activates the slotsync worker
```

```bash
# standby_2 postgresql.conf (failover candidate)
primary_conninfo = 'host=primary port=5432 user=replicator dbname=postgres application_name=standby_2'
primary_slot_name = 'phys_standby_2'
hot_standby_feedback = on
sync_replication_slots = on
```

```bash
# read_replica postgresql.conf (never promoted)
primary_conninfo = 'host=primary port=5432 user=replicator dbname=postgres application_name=read_replica'
# No primary_slot_name, no sync_replication_slots
# Not listed in synchronous_standby_names or synchronized_standby_slots
```

A few things worth noting in this setup. The `application_name` values in `primary_conninfo` on the standbys must match what you put in `synchronous_standby_names`. The physical slot names in `primary_slot_name` on the standbys must match what you put in `synchronized_standby_slots`. These are two separate namespaces and PostgreSQL will not cross-reference or validate them for you.

Also keep in mind that `synchronized_standby_slots` introduces a secondary shutdown behavior: the primary will not fully shut down until the listed standbys have confirmed receiving WAL up to the latest flushed position. This is intentional and protects slot consistency, but it is worth knowing upfront so a controlled shutdown of the primary does not catch you off guard.

## Conclusion

`synchronous_standby_names` and `synchronized_standby_slots` are both about making standbys confirm WAL, but the similarity ends there. One guards your transaction commits, the other guards your logical slot state. They operate independently, they identify standbys in different ways, and they can block completely different parts of your replication pipeline.

The practical takeaway is to configure each parameter with a specific, intentional standby list rather than assuming they should mirror each other. Only include standbys in `synchronized_standby_slots` that are genuine failover candidates, otherwise a lagging read replica can silently stall your logical subscribers. Never assume that synchronous replication alone makes your logical slots failover-safe. That requires the full slotsync setup to be in place.

Before deploying this configuration in production, it's always a good idea to verify the actual topology by checking `pg_replication_slots` and `pg_stat_replication` together to confirm which slots exist, which are synced, and which standbys are listed where. A quick audit there can save a lot of pain during an actual failover.

## References

- [PostgreSQL Documentation: Logical Decoding — Replication Slots Synchronization](https://www.postgresql.org/docs/current/logicaldecoding-explanation.html#LOGICALDECODING-REPLICATION-SLOTS-SYNCHRONIZATION)
- [PostgreSQL Documentation: synchronous_standby_names](https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-SYNCHRONOUS-STANDBY-NAMES)
- [idle_replication_slot_timeout parameter reference](https://postgresqlco.nf/doc/en/param/idle_replication_slot_timeout/)

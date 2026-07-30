+++
title = 'Upgrading a PostgreSQL Subscriber to Avoid Replaying WAL You Already Applied'
date = '2026-07-29'
draft = false
tags = ['postgresql', 'upgrade', 'pg_upgrade', 'logical-replication', 'replication', 'database', 'subscription', 'publication']
author = 'kylorend3r'
+++

**Summary:** Running `pg_upgrade` on a logical replication subscriber that's older than PostgreSQL 17 silently drops the subscription's replication origin — the bookmark tracking how far it has replayed. The publisher never finds out, so it replays WAL the subscriber already applied, surfacing as duplicate-key errors after an otherwise clean upgrade. This post covers why that happens, what PostgreSQL 17 changed to fix it, a manual procedure to protect subscribers on older versions, and the separate considerations for upgrading the publisher side, where publications survive the upgrade but replication slots never do.

## Table of Contents

- [Table of Contents](#table-of-contents)
- [The Post-Upgrade Problem](#the-post-upgrade-problem)
- [PostgreSQL Replication Process Tracking for Subscriptions (Beneath the Surface)](#postgresql-replication-process-tracking-for-subscriptions-beneath-the-surface)
- [The Role and Responsibility of pg\_upgrade](#the-role-and-responsibility-of-pg_upgrade)
- [Why the Publisher Replays WAL the Subscriber Already Applied](#why-the-publisher-replays-wal-the-subscriber-already-applied)
- [What Changed in PostgreSQL 17](#what-changed-in-postgresql-17)
- [The Safe Procedure for Pre-PG17 Subscribers](#the-safe-procedure-for-pre-pg17-subscribers)
  - [Step 1: Document and Disable Subscriptions Before the Upgrade](#step-1-document-and-disable-subscriptions-before-the-upgrade)
  - [Step 2: Capture the Resume LSN](#step-2-capture-the-resume-lsn)
  - [Step 3: Verify Every Table Is in a Ready State](#step-3-verify-every-table-is-in-a-ready-state)
  - [Step 4: After pg\_upgrade — Re-Point the Slot and Advance the Origin](#step-4-after-pg_upgrade--re-point-the-slot-and-advance-the-origin)
- [The Other Side: Upgrading a Publisher's Publications and Replication Slots](#the-other-side-upgrading-a-publishers-publications-and-replication-slots)
- [Key Takeaways](#key-takeaways)
- [Conclusion](#conclusion)
- [References](#references)

---

Upgrading a PostgreSQL cluster that is also a logical replication subscriber — an instance streaming data from another cluster via `CREATE SUBSCRIPTION` — is one of the less-documented ways `pg_upgrade` can hurt you. The schema comes over fine. The data comes over fine. The subscription comes back showing `enabled = true`. And then, intermittently, the apply worker throws duplicate-key violations on rows that were already replicated long before the upgrade started.

Nothing about the data was wrong. The subscriber just re-applied WAL it had already applied once. This post explains exactly why that happens, what changed in PostgreSQL 17 to fix part of it, and the manual procedure to protect a subscriber on any version before that.

This is the second post in a series on production PostgreSQL upgrades. The first post, [What It Actually Takes to Upgrade PostgreSQL in Production Without Breaking Everything](/posts/postgresql-major-upgrade-pg13-to-pg18-lessons-learned/), covered our PG13 → PG18 playbook end-to-end and two catalog-level bugs we hit along the way: a `NOT NULL` constraint name collision in `pg_restore`, and orphaned role OIDs in `pg_init_privs` after a role was dropped. That post touched logical replication only at the playbook level — disabling and re-enabling subscriptions around the upgrade window. It didn't cover what actually happens *inside* a subscriber's replication origin during that window, which is a big enough failure mode to deserve its own post. That's what this one is about.

## The Post-Upgrade Problem

The scenario looks like this: you run `pg_upgrade` on a PostgreSQL subscriber — say PG13 → PG18. The upgrade itself completes without error. `pg_subscription` shows the subscription with the right connection info and publication list. You re-enable it, the apply worker connects, replication resumes.

Then, minutes or hours later, the logs show something like:

```text
ERROR:  duplicate key value violates unique constraint "orders_pkey"
DETAIL:  Key (id)=(48213) already exists.
CONTEXT:  processing remote data for replication origin "pg_16512" during message type "INSERT" in transaction 884211, finished at 0/3A9F120
```

The row already existed. It was already replicated, correctly, before the upgrade window even started. The apply worker is not corrupting data — it is re-applying a slice of WAL it had already applied once, because it has no memory of ever having applied it.

## PostgreSQL Replication Process Tracking for Subscriptions (Beneath the Surface)

Every logical replication subscription is backed by a **replication origin** — an object that tracks how far replay has progressed for that specific subscription against that specific publisher. The PostgreSQL 18 documentation describes it directly:

> "The `pg_replication_origin_status` view contains information about how far replay for a certain origin has progressed."
> — PostgreSQL 18 docs, §53.19

![Replication origin tracking how far a subscription has replayed against its publisher](/images/posts/upgrading-postgresql-logical-replication-subscribers-with-pg-upgrade/replication_origin.png)

That progress marker is the origin's `remote_lsn` — the publisher-side LSN up to which this subscriber has confirmed applying changes. While the server is running, this value lives in shared memory, in an array PostgreSQL calls `replication_states`. The `pg_replication_origin_status` view queried above is just a thin wrapper around that array — every row it returns is read live from shared memory, never from disk.

That's precisely why `pg_logical/replorigin_checkpoint` has to exist at all: shared memory doesn't survive a restart on its own. `CheckPointReplicationOrigin()`, in `src/backend/replication/logical/origin.c`, dumps the `replication_states` array to that file during every regular checkpoint, purely so the value has somewhere durable to live between checkpoints and a crash or restart. At startup, PostgreSQL reads the file back and repopulates `replication_states` — only then does `pg_replication_origin_status` have anything to report again. The file is the origin's durability mechanism; the view is a window onto memory that the file exists to refill.

`CheckPointReplicationOrigin()` is defined in `origin.c`, but it doesn't run on its own — it's called from `xlog.c`, from a function named `CheckPointGuts()`. That name is literal: `CheckPointGuts()` is the actual body of I/O work a regular checkpoint performs, one subsystem at a time. It has no logic of its own beyond calling each subsystem's own checkpoint routine in sequence:

```c
static void
CheckPointGuts(XLogRecPtr checkPointRedo, int flags)
{
	CheckPointRelationMap();
	CheckPointReplicationOrigin();

	/* Write out all dirty data in SLRUs and the main buffer pool */
	TRACE_POSTGRESQL_BUFFER_CHECKPOINT_START(flags);
	CheckpointStats.ckpt_write_t = GetCurrentTimestamp();
	CheckPointCLOG();
	CheckPointCommitTs();
	CheckPointSUBTRANS();
	CheckPointMultiXact();
	CheckPointPredicate();
	CheckPointBuffers(flags);

	/* Perform all queued up fsyncs */
	TRACE_POSTGRESQL_BUFFER_CHECKPOINT_SYNC_START();
	CheckpointStats.ckpt_sync_t = GetCurrentTimestamp();
	ProcessSyncRequests();
	CheckpointStats.ckpt_sync_end_t = GetCurrentTimestamp();
	TRACE_POSTGRESQL_BUFFER_CHECKPOINT_DONE();
```

`CheckPointReplicationOrigin()` sits right alongside `CheckPointCLOG()`, `CheckPointCommitTs()`, and `CheckPointBuffers()` — the same calls responsible for flushing the commit log, commit timestamps, and dirty buffer pool pages to disk. The replication origin isn't special-cased or handled by some separate subscription-aware mechanism; it's checkpointed the same way, in the same pass, as every other piece of durable server state. That's the whole reason `pg_logical/replorigin_checkpoint` behaves like any other checkpoint artifact: it's written on the same cadence, by the same code path, as the rest of the cluster's crash-recovery state.

The key phrase there is *that same data directory*. The origin's durability guarantee was designed for restarts, crashes, and failovers within one running cluster — not for being physically relocated into a brand-new one.

## The Role and Responsibility of pg_upgrade

`pg_upgrade`'s job, at a mechanical level, is: dump the schema with `pg_dump --binary-upgrade`, restore it into a freshly `initdb`'d cluster, then physically link or copy the user table data files across. Two consequences fall directly out of that design:

1. `pg_logical/` in the new cluster is brand new — created by `initdb`. `pg_upgrade` never copies the old cluster's `pg_logical/replorigin_checkpoint` into it. There would be no reason to: the new cluster has entirely different internal object identities, so a raw copy of the old checkpoint file would be meaningless in the new one anyway.
2. Anything about subscription state that *does* survive the upgrade has to survive by being re-derived and re-emitted as SQL during the `pg_dump --binary-upgrade` step — not by physically carrying a file or a catalog row across in binary form.

That second point is the crux of the whole problem. Preserving origin progress across a major version upgrade was never a "copy the state" problem — it was an "emit SQL that reconstructs the state" problem. Nobody had written that SQL-emission logic before PostgreSQL 17.

On a source cluster older than PG17, none of the mechanisms that would make this safe exist:

- `pg_subscription_rel` isn't repopulated — the new subscription has no per-table sync state at all.
- The origin's `remote_lsn` isn't captured or restored — the new subscription's origin starts at `0/0`.
- There is no safety check that stops the upgrade even though the origin is about to be silently dropped.

The subscription metadata row in `pg_subscription` itself does come through, because it's part of the basic catalog dump. But it is a shell: correct connection info, correct publication list, and zero memory of how far replication had actually gotten.

## Why the Publisher Replays WAL the Subscriber Already Applied

When the subscription is re-enabled after the upgrade, the apply worker asks its origin where to resume. That origin is fresh — `0/0`. But here is the part that is easy to miss: the publisher does not take orders from the subscriber's origin at all.

Per the replication protocol (`doc/src/sgml/protocol.sgml`, `START_REPLICATION`), the publisher streams from `MAX(requested_lsn, slot's confirmed_flush_lsn)`. The slot's `confirmed_flush_lsn` is whatever the subscriber last confirmed — which could be from well before the upgrade window even started, since the slot itself is untouched by any of this. So the publisher happily replays a chunk of WAL the subscriber had already fully applied before the upgrade.

Rows that already exist get inserted again. Duplicate-key violations follow. Nothing was corrupted — the subscriber just lost its bookmark, and the publisher, having no reason to think anything changed, kept handing out WAL from where the slot says it left off.

![Diagram showing the publisher's slot confirmed_flush_lsn frozen behind the subscriber's previously-applied position, with the gap between them highlighted as replayed WAL that triggers duplicate-key violations](/images/posts/upgrading-postgresql-logical-replication-subscribers-with-pg-upgrade/lsn-replay-gap.svg)

*Before the upgrade, the slot and the origin are close together and agree on progress. After the upgrade, the origin resets to `0/0` but the slot's `confirmed_flush_lsn` never moved — so the publisher replays everything between the slot's frozen position and where the subscriber had actually already gotten to.*

## What Changed in PostgreSQL 17

PostgreSQL 17 closed this gap directly. When both the source and target clusters are ≥17.0, `pg_upgrade` now carries origin progress across as part of the binary-upgrade dump/restore sequence: `pg_subscription_rel` state is repopulated, the origin's `remote_lsn` is captured on the old cluster and restored on the new one, and a safety check aborts the upgrade if an origin's state can't be preserved rather than silently dropping it.

The important detail is that this gate is on **both sides**: the mechanism only fires when the cluster being upgraded is itself ≥17. A PG13 → PG18 upgrade of a subscriber does not benefit from this at all, because the *source* is nowhere near the 17.0 gate. The three protections above simply don't run, and you're back to the shell-row problem described above. This is exactly the situation that matters most in practice, since subscribers running old major versions are precisely the ones that most need the upgrade.

## The Safe Procedure for Pre-PG17 Subscribers

If your subscriber is on a version before 17, treat the subscription state as something you must manually preserve, not something `pg_upgrade` will carry for you.

![Diagram of the automated own-subscription upgrade flow: preflight health check, disabling the subscription and capturing its origin LSN to a JSON file in the upgrade phase, the pg_upgrade boundary the file survives, and the fixed re-enable order — reattach slot, restore origin, enable, refresh publication — in post_upgrade](/images/posts/upgrading-postgresql-logical-replication-subscribers-with-pg-upgrade/own-subscription-upgrade-flow.svg)

*The four steps below are exactly what this diagram shows end to end: disable and capture the origin's position before the upgrade, carry that value across the `pg_upgrade` boundary as the one thing that doesn't get regenerated, then re-attach, restore, enable, and refresh in that fixed order afterward.*

### Step 1: Document and Disable Subscriptions Before the Upgrade

Before touching anything, capture what exists so you have a record to rebuild from, then disable and detach the subscription's slot association:

```sql
-- Document your subscriptions
SELECT subname, subconninfo, subpublications, subenabled
FROM pg_subscription;

-- Document per-table sync state
SELECT s.subname, c.relname, r.srsubstate
FROM pg_subscription_rel r
JOIN pg_subscription s ON r.srsubid = s.oid
JOIN pg_class c ON r.srrelid = c.oid;

-- Disable the subscription
ALTER SUBSCRIPTION my_sub DISABLE;

-- Detach the subscription from its slot (decide separately whether
-- to keep or drop the remote slot on the publisher)
ALTER SUBSCRIPTION my_sub SET (slot_name = NONE);
```

### Step 2: Capture the Resume LSN

After disabling the subscription and confirming that its apply worker process has actually stopped on both the publisher and the subscriber, capture the origin's progress. This is the bookmark you will need to manually restore after the upgrade:

```sql
SELECT s.oid AS sub_oid,
       s.subname,
       o.external_id,
       pg_replication_origin_progress(o.external_id, true) AS resume_lsn
FROM pg_subscription s
JOIN pg_replication_origin_status o
  ON o.external_id = 'pg_' || s.oid::text
WHERE s.subname = 'mysub';
```

![psql output showing resume_lsn captured for a subscription via pg_replication_origin_progress before the upgrade](/images/posts/upgrading-postgresql-logical-replication-subscribers-with-pg-upgrade/capturing_resume_lsn.png)

Write `resume_lsn` down somewhere durable — outside the database you're about to upgrade. This single value is what prevents the duplicate-key scenario.

### Step 3: Verify Every Table Is in a Ready State

Before proceeding, confirm there is no table stuck mid-sync. A table in `srsubstate = 'i'` (initializing) or anything other than a settled state complicates the resume story, since its position isn't captured by the origin's single LSN the same way a steady-state table's is:

```sql
SELECT r.srrelid::regclass, r.srsubstate, r.srsublsn
FROM pg_subscription_rel r
JOIN pg_subscription s ON s.oid = r.srsubid
WHERE s.subname = 'mysub'
ORDER BY 1;
```

The states you want to see here are `r` (ready) — `i` (initializing) means it's still in the initial data copy, and stopping the upgrade mid-sync leaves that table in an ambiguous position.

### Step 4: After pg_upgrade — Re-Point the Slot and Advance the Origin

Once `pg_upgrade` has completed on the subscriber, the subscription row exists but its origin is at `0/0`. Re-point it to the correct slot, re-enable it, and manually advance the origin to the `resume_lsn` you captured in Step 2 — this is what tells the apply worker "you don't need to replay anything before this point, you already have it":

```sql
ALTER SUBSCRIPTION my_sub SET (slot_name = 'my_slot');
ALTER SUBSCRIPTION my_sub ENABLE;

SELECT pg_replication_origin_advance(
    'pg_' || (SELECT oid::text FROM pg_subscription WHERE subname = 'my_sub'),
    '0/3A4B5C0'::pg_lsn  -- the resume_lsn captured before the upgrade
);

ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION WITH (copy_data = false);

-- Confirm state
SELECT * FROM pg_subscription;
SELECT * FROM pg_subscription_rel;
```

`copy_data = false` here matters for the same reason it matters in any post-upgrade subscriber refresh: without it, PostgreSQL treats the refresh as an initial sync and re-copies full table data instead of resuming from the origin you just advanced.

Order matters in this sequence. `pg_replication_origin_advance` must run before the apply worker starts pulling WAL under the new origin, which is why the subscription is enabled first but the advance call happens immediately after, before any meaningful volume of WAL has a chance to flow.

## The Other Side: Upgrading a Publisher's Publications and Replication Slots

Everything so far has assumed the *subscriber* is the instance going through `pg_upgrade`. Logical replication topologies have two sides, though, and the *publisher* has its own upgrade considerations — separate from the origin problem above, and, importantly, not something the PostgreSQL 17 fix touches at all.

**Publications are preserved.** A `CREATE PUBLICATION` object lives in `pg_publication` as a plain catalog row. It carries no runtime replication-progress state — unlike a slot or an origin, it's just metadata describing which tables and operations are published. Because of that, publications come through the standard `pg_dump --binary-upgrade` schema dump cleanly, the same way a view or a function definition would. There is nothing special to do here: after upgrading a publisher, `SELECT * FROM pg_publication;` shows exactly what it did before.

**Logical replication slots are not preserved — on any version.** This is the part that trips people up, because it's tempting to assume the PG17 fix described earlier also covers this. It doesn't. A replication slot is runtime state tied to the specific WAL position and catalog `xmin` it was created against on the *old* cluster. `pg_upgrade` has never carried slots across, on any version, for either physical or logical slots. After upgrading a publisher, querying `pg_replication_slots` on the new cluster returns nothing for what used to be there — every downstream subscriber's slot reference is gone.

**Before the upgrade**, disable the subscriptions on every downstream subscriber first — the slot they depend on is about to disappear regardless, and leaving them enabled just produces connection errors during the window — then identify every logical slot on the publisher so you know what needs to come back:

```sql
-- On every downstream subscriber, before the publisher's upgrade window starts
ALTER SUBSCRIPTION my_sub DISABLE;

-- On the publisher
SELECT slot_name, plugin, database
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

**After the upgrade**, recreate the slots that are still needed:

```sql
-- On the publisher
SELECT pg_create_logical_replication_slot('your_slot_name', 'pgoutput');
```

Here is the step that is easy to get wrong: the recreated slot starts from the *current* WAL position on the new cluster. It does not, and cannot, resume from wherever the old slot's `confirmed_flush_lsn` was. Logical decoding depends on catalog history that was retained specifically because the old slot's `catalog_xmin` held it back from being vacuumed away — once that slot is gone, that guarantee is gone with it, regardless of whether the WAL segments themselves are still physically on disk.

This matters concretely because of the `MAX(requested_lsn, confirmed_flush_lsn)` rule from earlier in this post. If you simply re-point the subscription at the new slot and flip it back to `ENABLE` without an explicit resync, the apply worker requests its old resume position, the server clamps that up to the new slot's `confirmed_flush_lsn` (which is "now"), and replication resumes silently — skipping every change made between the last pre-upgrade sync and slot recreation. There is no error. No duplicate-key violation either. Just a quiet, permanent gap in the subscriber's data that nothing will flag on its own. The only safe way to re-enable is to force a resync explicitly:

```sql
-- On the subscriber, after the publisher's new slot exists
ALTER SUBSCRIPTION my_sub SET (slot_name = 'your_slot_name');
ALTER SUBSCRIPTION my_sub ENABLE;
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION WITH (copy_data = true);
```

`copy_data = true` here is the opposite of what Step 4 used on the subscriber-upgrade path, and deliberately so: there, the origin still had a valid bookmark to resume from and a re-copy would have been wasteful. Here, the bookmark is unrecoverable — the old slot that made it meaningful is gone — so `copy_data = true` is what prevents the silent gap. Plan for that resync window; it is not optional, and it cannot be replaced by the `resume_lsn` / `pg_replication_origin_advance` technique from the previous section, which only works when the publisher's slot is left untouched throughout.

## Key Takeaways

**Subscriber side:**

- Identify every subscription before the upgrade — subscription state, specifically origin progress, is not preserved by `pg_upgrade` on source clusters below PostgreSQL 17.
- Review `pg_subscription_rel` on the subscriber and confirm every relation is in `i` or `r` state before you disable subscriptions for the upgrade window.
- Capture the origin's `resume_lsn` via `pg_replication_origin_progress` before the upgrade — this is the one value that prevents duplicate-key violations afterward.
- Drop and re-create the slot association in a controlled sequence: disable and detach before, re-point and advance the origin after — never rely on `enabled = true` alone as a sign the subscription is safe to trust.
- If both the publisher and the subscriber are already on PostgreSQL 17 or later, this entire manual procedure is unnecessary — `pg_upgrade` carries origin state across natively.

**Publisher side:**

- Publications are preserved after the upgrade — no action needed, they come through the schema dump like any other catalog object.
- Replication slots are not preserved, on any PostgreSQL version. Identify them beforehand and recreate them afterward in a controlled manner.
- Disable subscriptions on every downstream subscriber before the publisher's upgrade window starts, not after — the slot is going away regardless, and this just avoids connection errors in the interim.
- Never re-enable a subscription against a recreated slot with a plain `ENABLE`. Without an explicit `REFRESH PUBLICATION WITH (copy_data = true)`, the apply worker silently resumes from the new slot's current position and quietly skips the entire gap — no error, no duplicate-key violation, just missing data.
- This resync requirement is unconditional for a publisher-side upgrade. It is not something the PostgreSQL 17 fix touches, and there is no `resume_lsn` equivalent that avoids it — that trick only applies when the publisher's slot is left untouched.

## Conclusion

The failure mode here is deceptive precisely because everything *looks* successful. The upgrade finishes cleanly, the schema is intact, the subscription shows `enabled`. The only sign anything is wrong is a wave of duplicate-key errors that starts sometime after replication resumes — by which point it is much harder to reconstruct what should have happened than it would have been to prevent it.

The root cause is not a bug so much as a structural gap: before PostgreSQL 17, nobody had written the logic to re-derive and re-emit replication origin state as SQL during a binary-upgrade dump. `pg_upgrade` physically relocates data files, but a replication origin's progress marker was never designed to be meaningful outside the data directory that checkpointed it. If your subscriber is on any version before 17, treat that origin LSN as something only you can carry across — document it, disable cleanly, and restore it explicitly once the new cluster is up.

If you are planning a logical replication upgrade of your own, a few things are worth deciding before you touch either cluster:

- **Check both endpoints' versions first.** If subscriber and publisher are both already on PostgreSQL 17 or later, `pg_upgrade` carries origin state across natively and the manual procedure in this post is unnecessary. The moment either side is below 17, assume nothing survives and plan for the manual steps.
- **Capture the origin's `resume_lsn` before you disable anything.** It costs one query and it is the single value that separates a clean cutover from a wave of duplicate-key errors afterward. Write it down outside the cluster you're about to upgrade.
- **Treat the subscriber and publisher upgrades as two separate problems.** A subscriber upgrade risks silently replaying WAL (loud failure: duplicate keys). A publisher upgrade risks silently losing a replication slot (quiet failure: a data gap with no error at all). The fix for one is not the fix for the other — don't assume `copy_data = false` is always correct just because it was correct on the subscriber side.
- **Never trust `enabled = true` as proof a subscription is healthy after an upgrade.** Confirm `pg_subscription_rel` shows every table in `r` (ready) state and that the origin's LSN matches what you expect, rather than relying on the subscription flag alone.
- **Rehearse the whole sequence on a staging clone first.** Both failure modes in this post are invisible until well after the upgrade window closes — a staging run with production-sized WAL volume is the only reliable way to see whether the resync window and resume LSN math actually hold up before you do it against production.

## References

- [PostgreSQL Documentation: Replication Progress Tracking (§53.19)](https://www.postgresql.org/docs/current/replication-origins.html)
- [PostgreSQL Documentation: Streaming Replication Protocol — START_REPLICATION](https://www.postgresql.org/docs/current/protocol-replication.html)
- [PostgreSQL Documentation: pg_upgrade](https://www.postgresql.org/docs/current/pgupgrade.html)
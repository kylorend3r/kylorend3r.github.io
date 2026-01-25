+++
title = 'Simple Timeouts have a Powerful Impact in Taming Idle in Transaction Exhaustion in PostgreSQL'
date = '2025-01-25'
draft = false
description = 'Exploring how timeout settings can effectively manage idle-in-transaction connections and prevent resource exhaustion in PostgreSQL'
tags = ['postgresql', 'timeouts', 'database', 'connections', 'performance', 'configuration']
author = 'kylorend3r'
+++

<!-- 
This post was migrated from Medium: 
https://demirhuseyinn-94.medium.com/simple-timeouts-have-a-powerful-impact-in-taming-idle-in-transaction-exhaustion-in-postgresql-20ca385df920

Copy the content from Medium and paste it below, converting to Markdown format as needed.
-->

## Introduction

Previously I discussed idle connections which are in zombie state. In other words, we discussed how to maintain connections which are idle and dead in the network layer. Visit the following blogpost to find out the idea and the history behind this post.

[Zombie Connections in PostgreSQL and TCP Keepalives](https://medium.com/@demirhuseyinn-94/zombie-connections-in-postgresql-and-tcp-keepalives-14fa5053a16f)

I wanted to write a follow-up blogpost to cover remaining timeout GUCs on PostgreSQL to manage client connections more efficiently. In other words, I wanted to highlight and provide use-case examples about a couple of timeout GUCs on PostgreSQL which can help us to improve database reliability. Keep in mid that, the GUCs I'm going to discuss will be used many different ways but my main purpose is to highlight their importance and provide follow-up examples.

Before diving into configurations it would be great to recap/revisit the other states of a connections, idle in transaction and active.

## Table of Contents

* [Defining Idle In Transaction Connections](#defining-idle-in-transaction-connections)
* [Managing Idle In Transaction Connections with idle_in_transaction_session_timeout](#managing-idle-in-transaction-connections-with-idle_in_transaction_session_timeout)
* [Preventing Long-Running Statements with statement_timeout](#preventing-long-running-statements-with-statement_timeout)
* [The Idle Session Timeout](#the-idle-session-timeout)
* [Conclusion](#conclusion)
* [References](#references)

## Defining Idle In Transaction Connections

In PostgreSQL, idle in transaction is a different state since it indicates that a client is waiting for another thing to complete transaction. For example, a client executes a query and waits before committing a transaction. We can query the state of this transaction and discover that it is in idle in transaction state.

In other words, it is the backend is in a transaction, but it is currently not doing anything and could be waiting for an input from the end user. Keep in mind that, the definition doesn't contain all technical details and it's a basic overview of the situation. Think of it like this: the client application has told PostgreSQL, "Okay, I'm starting a series of actions that should be treated as one unit," but then it has gone silent. The connection to the database remains open, and the transaction is still active in the background, but no further SQL statements are being sent or executed.


![Idle in transaction connections in PostgreSQL](/images/posts/simple-timeouts-idle-in-transaction-exhaustion-postgresql/idle_2_1.webp)

*Visualization of idle in transaction connections in PostgreSQL*

Idle in transaction connection can become headache if they are underestimated by application or database itself. Put another way: if we don't use proper timeouts these connections will consume and waste our available resources. As a result, we can suffer from performance or stability issues. When we nail the performance&stability issues we have to think about the following items.

* **Locks:** Idle in transaction connections hold locks on database objects which can cause lock contention and wait events due to locking.
* **Resource Consumption:** Even though idle, the session still consumes server resources (memory, connection slots).
* **Bloat:** Uncommitted transactions can prevent the vacuum process from reclaiming dead tuples, leading to table bloat over time.

## Managing Idle In Transaction Connections with idle_in_transaction_session_timeout

In PostgreSQL we can use `idle_in_transaction_session_timeout` GUC to define how many seconds will client/database wait to terminate a connection in idle in transaction state. For example, if the value of this GUC is 10 seconds any connection will be terminated if it has been in idle in transaction state.

We can manage this GUC at client level or PostgreSQL level. There is no proper way of managing this parameter but if we provide the GUC at PostgreSQL level it will affect all clients connecting database. We may have different applications/clients with different purposes connecting to database. It might be a better management example if this parameter managed at client level. If the idea is managing the timeout at database level, however, it's better to use PostgreSQL level settings.

If we have to change the GUC at PostgreSQL there are different ways of doing this but I want to show the simplest one.

```sql
benchmark_v1=# ALTER SYSTEM SET idle_in_transaction_session_timeout TO '15s';
ALTER SYSTEM
benchmark_v1=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

You can use `postgresql.conf` file and other methods if exits. I'll not dive into client level settings because it can vary.

The preceding approach will cover the cases if the connection which is in idle in transaction state exceeds X amount of time PostgreSQL/client will able to terminate the connection and cleanup the resources. What happens if the statement has been working for, let's assume, 24 hours ? In other words, do we want statements to run without timeout ?

## Preventing Long-Running Statements with statement_timeout

To prevent a statement running too long depending on our requirements we can use `statement_timeout` GUC. The timeout is measured from the time a command arrives at the server until it is completed by the server. If multiple SQL statements appear in a single simple-query message, the timeout is applied to each statement separately. In extended query protocol, the timeout starts running when any query-related message. Setting statement timeout at PostgreSQL configuration file might not be proper way of doing this since it affects all statements in database.

When `statement_timeout` is exceeded, PostgreSQL cancels the currently executing SQL statement, allowing the session to potentially continue with subsequent commands. In contrast, when `idle_in_transaction_session_timeout` is triggered, PostgreSQL takes the more drastic step of terminating the entire database session. This difference underscores their distinct purposes: one guards against runaway queries within an active session, while the other prevents inactive, open transactions from indefinitely holding resources. As for providing both GUCs simultaneously, they aren't mutually exclusive and are indeed used for different purposes. A client can set both, and PostgreSQL will enforce each independently according to its specific trigger condition.

```sql
benchmark_v1=# ALTER SYSTEM SET statement_timeout TO '60s';
ALTER SYSTEM
benchmark_v1=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

benchmark_v1=# show statement_timeout ;
 statement_timeout
-------------------
 1min
(1 row)
```

## The Idle Session Timeout

In PostgreSQL, there is another timeout option called `idle_session_timeout`. The GUC simply terminates any session that has been idle for longer than the specified amount of time. Idle session refers to a session waiting for a client query and it doesn't contain any open transactions. With the light of this information, we don't have to use this timeout if there is a specific information since idle sessions doesn't hold locks and they only consume limited amount of server resources. In addition to this, if we enable this timeout connection pooling components and client may experience unexpected connection termination issues and this can cause problems that are not predictable (maybe).

## Conclusion

In this blogpost I tried to cover basic client timeouts in PostgreSQL which can be useful at basic level. The purpose of this post is to provide a follow-up GUCs about timeouts for idle in transaction sessions. The first post was about terminating zombie/dead idle connections on PostgreSQL you can visit the post and read this blogpost to gain a wider insight about the issue.

There are different ways of managing timeouts we mentioned in this post I only tried to provide the idea behind them and a basic way of changing it. Ways of changing and managing configurations depend on your environment. In addition to this, there are more timeout GUCs but I wanted to highlight the ones related with statement/session.

Reach out to me on LinkedIn or connect on Twitter. May the Force guide us as we optimize our databases for greater efficiency.

Demir.

## References

* [PostgreSQL Documentation: statement_timeout](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-STATEMENT-TIMEOUT)
* [PostgreSQL Documentation: idle_session_timeout](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-IDLE-SESSION-TIMEOUT)

---

*Originally published on [Medium](https://demirhuseyinn-94.medium.com/simple-timeouts-have-a-powerful-impact-in-taming-idle-in-transaction-exhaustion-in-postgresql-20ca385df920)*

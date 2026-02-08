+++
title = 'Optimizing PostgreSQL Query Performance with Range Partitioning'
date = '2024-04-01'
draft = false
description = 'Learn how to optimize PostgreSQL query performance using range partitioning combined with hash partitioning to achieve predictable response times and reduced IOPS'
tags = ['postgresql', 'partitioning', 'performance', 'pg_partman', 'database']
author = 'kylorend3r'
+++

## Table of Contents

- [The Puzzle of Unpredictable Response Times](#the-puzzle-of-unpredictable-response-times)
- [The Challenge: Unpredictable Query Response Times](#the-challenge-unpredictable-query-response-times)
- [Initial Query and Performance Analysis](#initial-query-and-performance-analysis)
- [The Solution: Introducing Range Partitioning](#the-solution-introducing-range-partitioning)
- [Automating Sub-Partitioning with pg_partman Extension](#automating-sub-partitioning-with-pg_partman-extension)
- [Results: Improved Performance Metrics](#results-improved-performance-metrics)
- [The New Table Model: A Closer Look](#the-new-table-model-a-closer-look)
- [The Original Table Model: A Comparative Analysis](#the-original-table-model-a-comparative-analysis)
- [Decoding the Performance Delta](#decoding-the-performance-delta)
- [Unraveling the Impact](#unraveling-the-impact)
- [Conclusion](#conclusion)
- [References](#references)

If you've found yourself grappling with unpredictable query response times and sluggish SELECT queries in PostgreSQL, the solution might not just lie in reducing IOPS. There's a deeper, more technical approach that involves not only optimizing I/O operations but also distributing data more homogeneously in smaller, more manageable pieces. The key? Range partitioning.

## The Puzzle of Unpredictable Response Times

Imagine a scenario where you have a table named "ParcelData" that's partitioned by hash on the "ProviderID" column. However, there's a twist — some records, characterized by specific "ProviderID"s, contribute to a whopping 50% of the total table. This uneven distribution wreaks havoc on the predictability of response times for your SELECT queries, especially when archiving isn't in play.

So, the burning question is: How can you not only stabilize but supercharge your query performance in PostgreSQL?

## The Challenge: Unpredictable Query Response Times

Our initial setup involves a table named "ParcelData" partitioned by hash on the "ProviderID" column. A common challenge in this scenario is the uneven distribution of data. Some records, identified by a specific "ProviderID," contribute to 50% of the total table. This non-uniform distribution leads to unpredictable response times for SELECT queries, impacting overall performance.

Additionally, hash partitioning alone fails to stabilize response times, especially when archiving is not enabled. This becomes evident when executing a SELECT query without archiving, resulting in extended query durations.

## Initial Query and Performance Analysis

Let's consider an example SELECT query:

```sql
SELECT *
FROM "ParcelData"
WHERE "ProviderID" = '6362'
    AND "VerificationDateTime" > '2023-11-01 14:09:07.174611'
    AND "VerificationDateTime" < '2023-11-30 14:09:07.174611'
ORDER BY "ParcelID";
```

The query execution plan reveals a response time of approximately 50ms, indicating room for improvement.

## The Solution: Introducing Range Partitioning

To address the uneven data distribution and enhance query performance, we introduce range partitioning based on the "VerificationDateTime" column. This involves recreating the table schema and adding range partitions for each hash partition. The goal is to reduce the number of buffers accessed by PostgreSQL, subsequently decreasing required IOPS per query execution.

The modified table schema looks like this:

```sql
CREATE TABLE "ParcelData" (
    "ParcelID"                      BIGINT NOT NULL,
    "OrderID"                       BIGINT,
    "ConsignmentID"                 BIGINT,
    "ShipmentNumber"                BIGINT,
    "ProviderID"                    BIGINT,
    "Status"                        TEXT,
    "ItemQuantity"                  INTEGER,
    "CourierProviderID"             INTEGER,
    "PaymentStatus"                 INTEGER,
    "VerificationDate"              BIGINT,
    "VerificationDateTime"          TIMESTAMP WITHOUT TIME ZONE,
    "AgreedDeliveryDate"            BIGINT,
    "CancellationDate"              BIGINT,
    "CancellationDateTime"          TIMESTAMP WITHOUT TIME ZONE,
    "ShipmentDate"                  BIGINT,
    "ShipmentDateTime"              TIMESTAMP WITHOUT TIME ZONE,
    "DeliveryDate"                  BIGINT,
    "DeliveryDateTime"              TIMESTAMP WITHOUT TIME ZONE,
    "Created"                       TIMESTAMP WITHOUT TIME ZONE,
    "Updated"                       TIMESTAMP WITHOUT TIME ZONE,
    "UnDeliveredItemCount"          INTEGER,
    "AgreedDeliveryDateTime"        TIMESTAMP WITHOUT TIME ZONE,
    "CountryCode"                   TEXT,

    PRIMARY KEY ("ParcelID","ProviderID","VerificationDateTime")
) PARTITION BY HASH ("ProviderID");

-- Index creation
CREATE INDEX idx_parceldata_providerid_date_inc ON "ParcelData" ("ProviderID", "VerificationDateTime", "ParcelID") INCLUDE ("OrderID", "ConsignmentID", "ShipmentNumber", "Status", "ItemQuantity", "CourierProviderID", "PaymentStatus", "VerificationDate", "AgreedDeliveryDate", "CancellationDate", "CancellationDateTime", "ShipmentDate", "ShipmentDateTime", "DeliveryDate", "DeliveryDateTime", "Created", "Updated", "UnDeliveredItemCount", "AgreedDeliveryDateTime", "CountryCode");

CREATE INDEX idx_parceldata_providerid_status ON "ParcelData" ("ProviderID", "Status", "VerificationDateTime");

CREATE INDEX idx_parceldata_providerid_status_shipment_inc ON "ParcelData" ("ProviderID", "Status", "ShipmentDate") INCLUDE ("ShipmentDateTime", "CourierProviderID", "ItemQuantity");

-- Partition creation
CREATE TABLE "ParcelData_1" PARTITION OF "ParcelData"
FOR VALUES WITH (modulus 5, remainder 0) PARTITION BY RANGE ("VerificationDateTime");

CREATE TABLE "ParcelData_2" PARTITION OF "ParcelData"
FOR VALUES WITH (modulus 5, remainder 1) PARTITION BY RANGE ("VerificationDateTime");

CREATE TABLE "ParcelData_3" PARTITION OF "ParcelData"
FOR VALUES WITH (modulus 5, remainder 2) PARTITION BY RANGE ("VerificationDateTime");

CREATE TABLE "ParcelData_4" PARTITION OF "ParcelData"
FOR VALUES WITH (modulus 5, remainder 3) PARTITION BY RANGE ("VerificationDateTime");

CREATE TABLE "ParcelData_5" PARTITION OF "ParcelData"
FOR VALUES WITH (modulus 5, remainder 4) PARTITION BY RANGE ("VerificationDateTime");
```

## Automating Sub-Partitioning with pg_partman Extension

To streamline the management of sub-partitions, we leverage the pg_partman extension. This extension simplifies the creation and maintenance of time-based partitions. Here's a summary of the steps involved:

1. Install pg_partman extension if not already installed.

```bash
apt-get install postgresql-{pg_release}-partman
```

2. Create extension in the database.

```sql
ALTER SYSTEM SET shared_preload_libraries TO 'pg_partman_bgw';
```

3. Configure administration schema and permissions for partman.

```sql
CREATE SCHEMA partman;
CREATE EXTENSION pg_partman SCHEMA partman;
CREATE ROLE partman WITH LOGIN;
GRANT ALL ON SCHEMA partman TO partman;
GRANT ALL ON ALL TABLES IN SCHEMA partman TO partman;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA partman TO partman;
GRANT EXECUTE ON ALL PROCEDURES IN SCHEMA partman TO partman;
GRANT ALL ON SCHEMA public TO partman;
GRANT TEMPORARY ON DATABASE db_name to partman;
GRANT CREATE ON DATABASE db_name TO partman;
```

4. Create base tables for each hash partition.

5. Use pg_partman to create parent tables with range partitions.

The following commands illustrate the process:

```sql
-- Base table creation (e.g., ParcelData_1_base, ParcelData_2_base, ...)
CREATE TABLE "ParcelData_1_base" (LIKE "ParcelData_1");
-- Repeat for other base tables

-- Use pg_partman to create parent tables with range partitions
SELECT partman.create_parent(
    p_parent_table := 'public.ParcelData_1',
    p_control := 'VerificationDateTime',
    p_interval := '1 month',
    p_template_table := 'public.ParcelData_1_base',
    p_premake := 6
);
-- Repeat for other parent tables
```

## Results: Improved Performance Metrics

By combining hash and range partitioning, the query performance can be significantly enhanced. The optimized partitioning strategy helps in achieving the following goals:

- Decreased number of buffers (shared + hit buffers) accessed by PostgreSQL.
- Reduced required IOPS per query execution.

This results in an overall improvement in query execution times and a more predictable and stable performance for SELECT queries.

## The New Table Model: A Closer Look

```text
Index Scan using "ParcelData_1_p20231101_pkey" on "ParcelData_1_p20231101" "Packages"  (cost=0.29..3232.19 rows=39891 width=163) (actual time=0.016..16.453 rows=39889 loops=1)
  Index Cond: (("ParcelDataId" = '6362'::bigint) AND ("VerificationFriendlyDateTime" > '2023-11-01 14:09:07.174611'::timestamp without time zone) AND ("VerificationFriendlyDateTime" < '2023-11-30 14:09:07.174611'::timestamp without time zone))
  Buffers: shared hit=40124
Planning Time: 0.158 ms
Execution Time: 17.821 ms
```

In this new iteration, our query execution plan reveals an Index Scan using the "Packages_1_p20231101_pkey" on the "Packages_1_p20231101" partition. The critical factors here include a significantly reduced execution time of 17.821 ms and a noteworthy decrease in the number of shared buffers accessed by PostgreSQL (40124 buffers).

## The Original Table Model: A Comparative Analysis

```text
Sort  (cost=7776.80..7863.83 rows=34813 width=164) (actual time=38.355..42.354 rows=39889 loops=1)
  Sort Key: "ParcelData"."ParcelId"
  Sort Method: quicksort  Memory: 9951kB
  Buffers: shared hit=1333
  ->  Index Only Scan using "ParcelData_idx" on "ParcelData" "ParcelData"  (cost=0.42..5150.62 rows=34813 width=164) (actual time=0.019..13.278 rows=39889 loops=1)
        Index Cond: (("SupplierId" = '6362'::bigint) AND ("VerificationFriendlyDateTime" > '2023-11-01 14:09:07.174611'::timestamp without time zone) AND ("VerificationFriendlyDateTime" < '2023-11-30 14:09:07.174611'::timestamp without time zone))
        Heap Fetches: 5
        Buffers: shared hit=1333
Planning Time: 0.200 ms
Execution Time: 44.268 ms
```

In contrast, the original table model displays a Sort operation with a more extended execution time of 44.268 ms. The Index Only Scan on "Packages_1" involves accessing shared buffers (1333 buffers) and fetching from the heap.

## Decoding the Performance Delta

The numbers speak for themselves. The new table model showcases a remarkable improvement in query performance, responding in approximately 17 ms. This is in stark contrast to the original model, which registers a response time of around 44 ms.

## Unraveling the Impact

The key to this performance delta lies in the strategic application of range partitioning. By breaking down the data into smaller, more homogeneous subsets based on the "VerificationDateTime" column, our queries navigate through a more streamlined and efficient data structure.

The new model not only reduces IOPS but also enables faster search operations. The combination of hash and range partitioning creates a synergy that optimizes query execution, resulting in a more predictable and speedier response. The new model exhibits a staggering 59.77% reduction in query execution time compared to the original model. This percentage gain emphasizes the transformative impact of nuanced database architecture on system efficiency. It's not just about reducing IOPS; it's about unleashing the full potential of your PostgreSQL database.

## Conclusion

While the presented results and methods showcase the transformative potential of PostgreSQL partitioning, their relevance hinges on your database's requirements and expectations. Before embarking on partitioning endeavors, evaluate your database landscape, define performance benchmarks, and tailor your strategies accordingly. It's not about adopting partitioning for the sake of it; it's about sculpting your database architecture to meet the unique demands of your applications and workloads.

As you explore the possibilities of PostgreSQL partitioning, remember: the success of these strategies lies in their alignment with your database objectives. Embrace a tailored approach, evaluate your requirements, and adapt these methods to unleash the full potential of your database architecture.

If you have questions or feedbacks please feel free to each out to me on [LinkedIn](https://linkedin.com), connect on [Twitter](https://twitter.com) or [Superpeer](https://superpeer.com).

## References

- [pg_partman GitHub Repository](https://github.com/pgpartman/pg_partman)
- [AWS RDS PostgreSQL Partitions Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL_Partitions.html)
- [pg_partman Documentation](https://github.com/pgpartman/pg_partman/blob/master/doc/pg_partman.md#maintenance-functions)

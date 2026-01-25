+++
title = 'PostgreSQL Configuration Chronicles: Memory Related Configurations'
date = '2024-04-24'
draft = false
description = 'Exploring PostgreSQL memory-related parameters and their impact on system performance, with practical benchmarking and configuration guidance'
tags = ['postgresql', 'memory', 'database', 'performance', 'configuration', 'shared_buffers', 'work_mem']
author = 'kylorend3r'
+++

## Introduction

We explore the fundamental role memory plays within the architecture of PostgreSQL databases, shedding light on the intricacies of memory-related parameters and their profound impact on system performance and resource utilization. Our journey begins with an exploration of the memory architecture of PostgreSQL, uncovering the underlying mechanisms that govern memory allocation and usage within the database engine. From there, we'll study on the crucial memory-related parameters that administrators must navigate to fine-tune their PostgreSQL deployments for optimal performance. We'll dissect each parameter, discussing its significance and the potential ramifications of setting values too low or too high. We'll guide you through the intricate process of configuring these parameters, providing step-by-step instructions to ensure your PostgreSQL instance is finely tuned to meet the demands of your workload. And finally, we'll bring theory into practice, embarking on a journey of benchmarking and analysis to uncover the real-world impact of these memory parameters on performance and resource utilization.

## Table of Contents

* [Memory Related Parameters in PostgreSQL](#memory-related-parameters-in-postgresql)
* [Configuring Memory Parameters](#configuring-memory-parameters)
* [Benchmark Tests and Results](#benchmark-tests-and-results)
* [Conclusion](#conclusion)

## Memory Related Parameters in PostgreSQL

**`shared_buffers`:** Determines PostgreSQL's cache memory. Too low a value can increase disk I/O, hampering performance, while too high a value may starve the OS and other processes of memory.

**`work_mem`:** Controls memory for individual operations within queries. Too low can lead to inefficient query plans, while too high may exhaust memory, especially under heavy concurrent query loads.

**`hash_mem_multiplier`:** Multiplies `work_mem` for hash tables, impacting hash join performance. Too low a value can result in inefficient joins, while too high can lead to excessive memory usage.

**`effective_cache_size`:** Estimates system memory available for disk caching. Setting it too low may underutilize available memory, increasing disk I/O, while setting it too high may lead to inefficient query plans.

**`max_connections`:** Limits concurrent database connections. Too low a value can cause connection errors during peak times, while too high can strain system resources.

**`maintenance_work_mem`:** Allocates memory for maintenance tasks like vacuuming. Too low can prolong maintenance operations, while too high can lead to memory contention.

**`autovacuum_work_mem`:** Controls memory for autovacuum processes. Too low may result in ineffective vacuuming, while too high can impact system performance due to excessive memory usage.

**`vacuum_buffer_usage_limit`:** Sets a limit on memory used by vacuum operations. Too low a value may decrease vacuum effectiveness, while too high can cause memory contention.

**`logical_decoding_work_mem`:** Allocates memory for logical decoding, crucial for replication. Too low can degrade replication performance, while too high can affect memory availability for other operations.

## Configuring Memory Parameters

Configuring these parameters effectively is crucial for PostgreSQL performance. Consider workload characteristics, available system resources, and concurrent operations when setting values. Regular monitoring and tuning are essential to maintain optimal performance over time. Experiment cautiously with values, testing performance impact under realistic conditions before applying changes in production environments.

## Benchmark Tests and Results

To compare the performance of two PostgreSQL instances, both hosting identical databases with the same table and size, I employed the popular benchmarking tool pgbench. Initially, I initialized the databases with a scale factor of 250 using the following command:

```bash
pgbench -i -s 250 benchmark
```

```bash
pgbench -S -v -c 200 -j 2 -T 600 benchmark >> /tmp/results.txt
```

This command runs the pgbench tool with the following options:

* **-S:** Runs in simple mode, which means it uses a single SELECT statement as its benchmark transaction.
* **-v:** Enables verbose mode, so it will output more detailed information about the benchmark as it runs.
* **-c 200:** Specifies the number of client connections to simulate. In this case, it will simulate 200 client connections.
* **-j 2:** Specifies the number of threads to use. It will use 2 threads.
* **-T 600:** Specifies the duration of the test in seconds. It will run the benchmark for 600 seconds (or 10 minutes).
* **benchmark:** Specifies the name of the database to benchmark.
* **>> /tmp/results.txt:** Redirects the output of the benchmark to a file named results.txt in the /tmp directory. The >> operator appends the output to the file if it already exists or creates it if it doesn't exist.

Following this, I sought to enhance the performance of one PostgreSQL instance by adjusting several key configuration parameters. These adjustments were made with the expectation of achieving better performance:

```sql
ALTER SYSTEM SET shared_buffers TO '4GB';
ALTER SYSTEM SET work_mem TO '8MB';
ALTER SYSTEM SET hash_mem_multiplier TO 3;
ALTER SYSTEM SET effective_cache_size TO '8GB';
ALTER SYSTEM SET maintenance_work_mem TO '1GB';
ALTER SYSTEM SET autovacuum_work_mem TO '1GB';
ALTER SYSTEM SET logical_decoding_work_mem TO '128MB';
```

Surprisingly, the benchmark results revealed that the tweaked PostgreSQL instance consumed more memory than the default one. This observation underscores the fact that increasing cache and memory limits for PostgreSQL correlates with higher memory consumption. As a result, it becomes imperative to diligently analyze and monitor system resources to configure these limits optimally, ensuring efficient resource utilization without sacrificing performance.

![Default PostgreSQL Instance Memory Usage - 3.5GB](/images/posts/postgresql-configuration-chronicles-memory-related-configurations/memory_default_3_5gb.webp)

*Default PostgreSQL Instance Memory Usage - 3.5GB*

![Optimized PostgreSQL Instance Memory Usage - 4.5GB](/images/posts/postgresql-configuration-chronicles-memory-related-configurations/memory_optimized_4_5gb.webp)

*Optimized PostgreSQL Instance Memory Usage - 4.5GB*

Despite the increased memory consumption, the optimized configuration yielded significant improvements in latency, particularly in read-only workloads. With latency improvements ranging from 10% to 20%, the optimized PostgreSQL instance exhibited heightened responsiveness and efficiency in handling read-heavy workloads. This enhancement in latency not only translates to a smoother user experience but also contributes to overall system efficiency and productivity.

![Performance Comparison](/images/posts/postgresql-configuration-chronicles-memory-related-configurations/config.webp)

Finally, in terms of vacuum performance, the tweaked PostgreSQL instance outperformed its default counterpart by completing vacuum full in 28 seconds, compared to 32 seconds for the default configuration. To quantify this improvement, we can calculate the percentage reduction in vacuum full completion time:

**Percentage Improvement = ((32 - 28) / 32) × 100 = 12.5%**

This calculation reveals a notable improvement of approximately 12.5% in vacuum performance, further emphasizing the efficacy of the optimized PostgreSQL configuration in enhancing database maintenance operations.

![Vacuum Performance Comparison](/images/posts/postgresql-configuration-chronicles-memory-related-configurations/vacuum_performance.webp)

*Vacuum Performance Comparison*

## Conclusion

In conclusion, the crux of PostgreSQL memory optimization lies in understanding your database's unique needs. Instead of blindly adjusting parameters, assess system resources, query performance, and database characteristics beforehand. This ensures that changes don't just boost performance metrics but also optimize resource usage and stability. Finding the right balance is key — excessive memory can strain resources, while conservative settings may underutilize them. By analyzing and monitoring systematically, you can tailor configurations to your database, ensuring optimal performance without sacrificing stability.

Demir.

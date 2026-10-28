---
title: Partitioning
source: https://medium.com/@geekycodes/partitioning-in-apache-spark-how-to-control-parallelism-and-improve-performance-559684db7904
author:
published:
created: 2026-10-29
description: If you’ve followed this Apache Spark series so far, you’ve already learned about Spark Architecture, Driver vs Executor, DAGs, Stages…
tags:
  - Spark
---
> ***How does Spark divide a massive dataset so that multiple executors can process it in parallel?***

The answer is **partitioning**.

Partitioning is one of the most important concepts for anyone working with **PySpark, Databricks, Delta Lake, or large-scale ETL pipelines**.

A good partitioning strategy can improve:

- Parallelism
- CPU utilization
- Query performance
- Join performance
- Aggregation performance
- File-writing performance

A poor partitioning strategy can cause:

- Data skew
- Large partitions
- Small files
- Excessive shuffle
- Executor memory pressure
- Long-running tasks
- Poor cluster utilization
- Higher Databricks costs

### 1\. What Is a Partition in Spark?

A **partition** is a logical chunk of a distributed dataset.

Instead of processing a massive dataset as one unit, Spark divides it into multiple partitions.

For example, suppose we have:

```
1 billion records
```

Spark could divide the data into:

```
Partition 1 → 100 million records

Partition 2 → 100 million records

Partition 3 → 100 million records

...

Partition 10 → 100 million records
```

These partitions can then be processed in parallel.

Conceptually:

```
Dataset

                   |

        ┌──────────┼──────────┐

        ↓          ↓          ↓

   Partition 1  Partition 2  Partition 3

        ↓          ↓          ↓

    Executor    Executor    Executor
```

The important distinction is:

> ***Partition = unit of distributed data processing.***

### 2\. Why Does Spark Use Partitions?

Imagine trying to process a 5 TB dataset on a cluster.

You don't want a single machine to process all 5 TB.

Instead, Spark distributes the workload:

```
5 TB Dataset

     ↓

Partitions

     ↓

Multiple Executors

     ↓

Parallel Processing
```

This allows Spark to take advantage of the compute resources available in the cluster.

For example:

```
Executor 1 → Partition 1

Executor 2 → Partition 2

Executor 3 → Partition 3

Executor 4 → Partition 4
```

Multiple tasks can execute simultaneously.

This is one of the fundamental ideas behind Spark's distributed execution model.

**Related:** \[Spark Architecture →\]

### 3\. Partition vs Task

These two concepts are frequently confused in interviews.

A **partition** is a chunk of data.

A **task** is the unit of work that processes a partition.

For example:

```
100 partitions
```

may result in approximately:

```
100 tasks
```

for a particular stage.

Conceptually:

```
Partition 1 → Task 1

Partition 2 → Task 2

Partition 3 → Task 3

...

Partition 100 → Task 100
```

Those tasks are scheduled across available executor cores.

So remember:

> ***Partition = data*** \_ \_ ***Task = computation performed on that partition***

**Related:** \[Stages and Tasks in Apache Spark →\]

### 4\. How Many Partitions Should You Have?

There is no universal number.

This is one of the most important things to understand.

Suppose you have:

```
1 GB
```

You cannot simply say:

> *"Use 10 partitions."*

The right number depends on factors such as:

- Dataset size
- Number of CPU cores
- Type of operation
- Data distribution
- Shuffle volume
- Executor memory
- Input file sizes
- Downstream workload

The objective is to create enough partitions to keep the cluster busy without creating excessive scheduling overhead or tiny tasks.

### 5\. Too Few Partitions

Suppose your cluster has:

```
20 executor cores
```

but your dataset contains only:

```
4 partitions
```

Only approximately four tasks can be active for a stage at a time.

You may have unused compute capacity.

```
20 cores

████  ← 4 active tasks

                ↓

        16 cores underutilized
```

This can reduce parallelism.

### 6\. Too Many Partitions

Now consider the opposite.

Suppose you have:

```
100 MB
```

and create:

```
10,000 partitions
```

You could end up with thousands of tiny tasks.

That introduces overhead from:

- Task scheduling
- Task startup
- Metadata
- File management
- Excessive small output files

The goal is not:

> *"Maximum number of partitions."*

The goal is:

> ***Appropriate partition size and sufficient parallelism.***

### 7\. Inspecting the Number of Partitions

For an RDD:

```
df.rdd.getNumPartitions()
```

For example:

```
print(df.rdd.getNumPartitions())
```

You might get:

```
200
```

meaning the underlying RDD representation contains 200 partitions.

For DataFrame workloads, the exact physical partitioning can change throughout execution, particularly around shuffle stages.

That's why looking at the **Spark UI and physical execution plan** is often more useful than relying on a single partition count.

### 8\. repartition()

One of the most commonly used partitioning functions is:

```
df.repartition()
```

For example:

```
df = df.repartition(100)
```

This asks Spark to redistribute the DataFrame into 100 partitions.

Conceptually:

```
Current Data

     ↓

10 partitions

     ↓

Shuffle

     ↓

100 partitions
```

Because data generally has to move between partitions, `repartition()` usually causes a shuffle.

### 9\. repartition() by Column

You can also partition by a column:

```
df = df.repartition(

    "customer_id"

)
```

Now Spark can redistribute records according to `customer_id`.

You can also specify both the number of partitions and partitioning column:

```
df = df.repartition(

    200,

    "customer_id"

)
```

Conceptually:

```
customer_id

     ↓

Partitioning Function

     ↓

Partition 1

Partition 2

Partition 3

...

Partition 200
```

This can be useful when downstream operations frequently group or join using the same key.

However, it can also introduce a large shuffle.

### 10\. The Problem With repartition()

Consider:

```
df = df.repartition(

    500,

    "customer_id"

)
```

If the DataFrame is already appropriately distributed, this may introduce unnecessary work.

Spark has to redistribute the data.

Therefore, don't use:

```
repartition()
```

simply because:

> *"More partitions must mean more performance."*

That's not necessarily true.

Before repartitioning, ask:

1. Why do I need to repartition?
2. How large is the dataset?
3. How many cores are available?
4. Is the data skewed?
5. What operation comes next?
6. Will the shuffle actually improve the workload

### 11\. coalesce()

Another important function is:

```
df.coalesce()
```

It is primarily used to **reduce the number of partitions**.

For example:

```
df = df.coalesce(20)
```

Conceptually:

```
100 partitions

      ↓

  Coalesce

      ↓

20 partitions
```

In the common case of reducing partitions, `coalesce()` can avoid a full shuffle.

This makes it useful when you want to reduce the number of partitions after filtering or before writing.

### 12\. repartition() vs coalesce()

This is one of the most common Spark interview questions.

![None](https://freedium-mirror.cfd/img/medium/4000/1*q5-mH7G-CdleMcyDUihj-g.png)

Example:

```
df.repartition(100)
```

and:

```
df.coalesce(10)
```

are not equivalent operations.

A useful rule:

> Use `repartition()` when you need redistribution. Use `coalesce()` when you primarily need to reduce partitions without a full shuffle.

### 13\. Partitioning and Shuffle

Partitioning is closely connected to the concept of shuffle.

Suppose:

```
df.groupBy("customer_id").count()
```

Spark needs records with the same `customer_id` to be processed together.

This can require a shuffle.

```
Before:

P1 → customer 101, 102

P2 → customer 101, 103

P3 → customer 102, 103

             ↓

          Shuffle

             ↓

P1 → customer 101, 101

P2 → customer 102, 102

P3 → customer 103, 103
```

Understanding this relationship is critical:

```
Partitioning

     ↓

Data Distribution

     ↓

Shuffle

     ↓

Task Performance
```

**Related:** \[Shuffle in Apache Spark →\]

### 14\. Partitioning and Data Skew

Partitioning becomes especially important when the data is skewed.

Suppose:

```
Customer A → 30% of all transactions

Customer B → 0.01%

Customer C → 0.01%

...
```

If we partition by:

```
customer_id
```

we may get:

```
Partition 1 → 30% of data

Partition 2 → 5%

Partition 3 → 4%

Partition 4 → 3%

...
```

One task may therefore take significantly longer.

```
Task 1 → 3 minutes

Task 2 → 3 minutes

Task 3 → 4 minutes

Task 4 → 40 minutes
```

The entire stage can be delayed by the slow task.

This is called a **straggler**.

**Related:** \[Data Skew in Apache Spark →\]

### 15\. Partition Count Doesn't Fix Data Skew

This is a very important interview concept.

Suppose you have:

```
100 partitions
```

and one customer represents:

```
30% of the dataset
```

Increasing the number of partitions to:

```
500
```

doesn't necessarily solve the fundamental problem.

If all records for the same key are routed to the same partition, the hot key can still create a disproportionately large partition.

```
customer_id = 1001

        ↓

Same partition

        ↓

Huge amount of data
```

This is why skew often requires techniques such as:

- Salting
- AQE skew join handling
- Broadcast joins
- Pre-aggregation
- Alternative partitioning strategies

### 16\. Choosing a Partition Column

This is a common interview question:

> ***How do you choose a partition column?***

The answer depends on what you mean by partitioning.

For **execution partitioning**, consider:

- Data distribution
- Cardinality
- Query patterns
- Join keys
- Aggregation keys
- Skew
- Dataset size

For **storage partitioning**, additional considerations include:

- Common filter predicates
- Partition cardinality
- File sizes
- Query access patterns
- Number of resulting directories/files

Do not choose a column simply because it has the highest number of distinct values.

For example:

```
Column A → 1,000 distinct values

Column B → 100 distinct values

Column C → 10 distinct values
```

It is not automatically correct to choose Column A.

You need to consider how the data is distributed and how the table will actually be queried.

### 17\. High Cardinality Isn't Always Better

Consider:

```
user_id
```

with:

```
100 million unique users
```

It has very high cardinality.

Does that automatically make it a good storage partition column?

No.

You could end up with an enormous number of partitions/directories or small files.

A column such as:

```
event_date
```

might be much more practical for storage partitioning if queries commonly filter by date.

For example:

```
event_date=2026-09-20

event_date=2026-09-21

event_date=2026-09-22
```

The right choice depends on workload characteristics.

### 18\. Execution Partitioning vs Storage Partitioning

This distinction is extremely important.

### Execution Partitioning

Controls how Spark distributes data during computation.

Examples:

```
df.repartition(100)
```

or:

```
df.repartition("customer_id")
```

### Storage Partitioning

Controls how data is physically organized in storage.

For example, a Delta table might be organized by:

```
event_date
```

resulting conceptually in:

```
table/

├── event_date=2026-09-20/

├── event_date=2026-09-21/

└── event_date=2026-09-22/
```

These are related concepts, but they are **not the same thing**.

### 19\. Partitioning When Writing Data

Consider:

```
df.write \

  .partitionBy("event_date") \

  .parquet("/data/events")
```

Spark will organize the output by the partition column.

Conceptually:

```
/data/events/

event_date=2026-09-20/

    files...

event_date=2026-09-21/

    files...

event_date=2026-09-22/

    files...
```

This can be beneficial when queries frequently filter by:

```
WHERE event_date = '2026-09-22'
```

because the storage layout can help Spark avoid reading irrelevant partitions.

However, excessive partitioning can create too many small files.

### 20\. Partition Pruning

Suppose your table is partitioned by:

```
event_date
```

and your query is:

```
SELECT *

FROM events

WHERE event_date = '2026-09-22'
```

Spark can potentially read only the relevant partition rather than scanning all date partitions.

Conceptually:

```
10 years of data

      ↓

Filter on event_date

      ↓

Read relevant partition(s)

      ↓

Less data scanned
```

This is known as **partition pruning**.

Partition pruning is one of the major reasons storage partitioning can improve query performance.

### 21\. Don't Over-Partition Storage

Consider:

```
1 million records
```

and partitioning by:

```
user_id
```

where there are:

```
500,000 users
```

You could create a huge number of tiny partitions/files.

This is usually undesirable.

A better storage partition column often has:

- Reasonable cardinality
- Strong filtering value
- Good query alignment
- Sufficient data per partition

For example:

```
event_date
```

is often more practical than:

```
user_id
```

for time-series workloads.

### 22\. Partition Size Matters

The number of partitions alone isn't enough.

You should also consider:

> ***How much data does each partition contain?***

Suppose:

```
1 TB dataset
```

with:

```
10 partitions
```

That's roughly:

```
100 GB per partition
```

which may result in large tasks and memory pressure.

Now suppose:

```
1 TB
```

is split into:

```
100,000 partitions
```

You may create enormous scheduling and file overhead.

The objective is balanced partition sizes.

```
Too few

   ↓

Huge partitions

   ↓

Slow tasks

Too many

   ↓

Tiny partitions

   ↓

Scheduling overhead

Balanced

   ↓

Good parallelism
```

### 23\. Partitioning and Databricks Cost

Partitioning also affects cloud costs.

Suppose a poorly partitioned job causes:

```
Longer execution

+

More shuffle

+

More spill

+

Underutilized executors
```

The cluster may need to run significantly longer.

For a large Databricks ETL pipeline, that translates into increased compute cost.

Therefore:

> ***Partitioning is not just a performance concern. It can also be a cost optimization concern.***

This is particularly important for large-scale workloads running continuously.

### 24\. Using Spark UI to Diagnose Partition Problems

When troubleshooting a Spark application, open the **Spark UI** and inspect the stages.

Look for:

- Number of tasks
- Task duration
- Input size
- Shuffle read
- Shuffle write
- Spill
- Uneven task durations

Suppose you see:

```
Task 1 → 4 sec

Task 2 → 5 sec

Task 3 → 4 sec

Task 4 → 5 sec

Task 5 → 80 sec
```

That should immediately trigger an investigation.

Potential causes include:

- Data skew
- Uneven partition sizes
- Expensive records
- Executor problems
- External I/O

The Spark monitoring documentation explains how to inspect application and stage-level metrics through the Spark UI. [Apache Spark Monitoring Documentation](https://spark.apache.org/docs/latest/monitoring.html?utm_source=chatgpt.com)

### 25\. Partitioning With Adaptive Query Execution

Modern Spark includes **Adaptive Query Execution (AQE)**.

AQE can use runtime statistics to optimize execution.

For example, Spark may dynamically coalesce post-shuffle partitions when the initial partition count produces many small partitions.

Conceptually:

```
Initial Plan

     ↓

Execute

     ↓

Runtime Statistics

     ↓

AQE

     ↓

Adjust Partitioning

     ↓

Improved Execution
```

This is why modern Spark optimization isn't simply about manually choosing every partition count.

You should understand both:

- Static partitioning decisions
- Runtime optimization through AQE

[Apache Spark SQL Performance Tuning — Adaptive Query Execution](https://spark.apache.org/docs/latest/sql-performance-tuning.html?utm_source=chatgpt.com)

### 26\. A Practical PySpark Example

Suppose we have:

```
transactions = spark.read.parquet(

    "/data/transactions"

)
```

We want to aggregate transactions by customer.

A straightforward implementation is:

```
result = transactions.groupBy(

    "customer_id"

).sum("amount")
```

For a large dataset, we might investigate:

```
print(

    transactions.rdd.getNumPartitions()

)
```

Then inspect the Spark UI to understand:

- Input size
- Shuffle size
- Task distribution
- Skew
- Spill

If we determine that a specific partitioning strategy is useful for the workload, we could explicitly repartition:

```
transactions = transactions.repartition(

    200,

    "customer_id"

)
```

But we should not automatically assume this is faster.

The correct workflow is:

```
Measure

   ↓

Identify bottleneck

   ↓

Understand data distribution

   ↓

Change partitioning

   ↓

Measure again
```

### 27\. A Common Partitioning Mistake

One of the most common mistakes is:

```
df = df.repartition(1000)
```

followed by:

```
df = df.repartition(500)
```

followed by:

```
df = df.repartition(200)
```

throughout a pipeline.

Each redistribution can introduce additional shuffle.

Instead, ask:

> ***Does this repartition actually solve a downstream problem?***

If not, remove it.

Performance optimization should be evidence-driven.

### 28\. Partitioning Best Practices

Here are practical rules to remember.

### 1\. Don't blindly increase partitions

More partitions do not automatically mean better performance.

### 2\. Avoid unnecessary repartition()

Every redistribution has a cost.

### 3\. Use coalesce() when appropriate

Especially when reducing partitions without requiring a full redistribution.

### 4\. Watch for skew

Uneven data distribution can dominate execution time.

### 5\. Consider downstream operations

Partitioning should support the workload that follows.

### 6\. Don't confuse execution and storage partitioning

They solve different problems.

### 7\. Avoid excessive storage partitioning

High-cardinality columns can create too many files or directories.

### 8\. Use Spark UI

Measure actual task and partition behavior.

### 9\. Leverage AQE

Modern Spark can dynamically optimize parts of the execution plan.

### 10\. Optimize based on evidence

Don't optimize partition counts based purely on rules of thumb.

### 29\. Common Partitioning Interview Questions

### Q1. What is a partition in Spark?

A partition is a logical chunk of distributed data processed by Spark.

### Q2. What is the relationship between partitions and tasks?

A task generally processes one partition for a particular stage.

### Q3. What does repartition() do?

It redistributes data into the requested number of partitions and generally causes a shuffle.

### Q4. What does coalesce() do?

It reduces the number of partitions and can avoid a full shuffle when reducing partitions.

### Q5. Can repartition() increase partitions?

Yes.

```
df.repartition(200)
```

### Q6. Can coalesce() increase partitions?

No. It is intended for reducing partitions.

No.

Too many partitions can create scheduling and small-file overhead.

### Q8. How do you choose a partition column?

Consider cardinality, data distribution, skew, query patterns, joins, aggregations, and workload requirements.

### Q9. What happens if one partition is much larger than the others?

That can create a straggler task and increase overall stage execution time.

### Q10. How do you identify partition problems?

Use Spark UI, execution plans, task duration, input sizes, shuffle metrics, and spill metrics.

### Q11. What is partition pruning?

It is the ability to avoid scanning irrelevant storage partitions when query predicates match the partitioning scheme.

### Q12. Is partitioning the same as bucketing?

No. They are different physical data-layout concepts with different use cases.

### 30\. Partitioning: The Mental Model

Keep this diagram in mind:

```
DATASET

                    |

          ┌─────────┼─────────┐

          ↓         ↓         ↓

       Partition  Partition  Partition

          1         2         3

          |         |         |

          ↓         ↓         ↓

        Task      Task      Task

          |         |         |

          └─────────┼─────────┘

                    ↓

                Executors
```

And when redistribution is required:

```
Partitions

    ↓

   SHUFFLE

    ↓

New Partition Distribution

    ↓

Tasks

    ↓

Executors
```

The key principle is:

> ***Good partitioning allows Spark to keep the cluster busy without creating unnecessary data movement or overhead.***

### Key Takeaways

You should now understand:

- A partition is a chunk of distributed data.
- Tasks process partitions.
- Partitions enable parallel processing.
- `repartition()` generally causes a shuffle.
- `coalesce()` can reduce partitions without a full shuffle in the common case.
- More partitions do not automatically mean better performance.
- Too few partitions can limit parallelism.
- Too many partitions can create scheduling and small-file overhead.
- Data skew can create uneven partition sizes.
- Execution partitioning and storage partitioning are different concepts.
- Storage partitioning can enable partition pruning.
- AQE can dynamically optimize partitioning in some situations.
- Spark UI is essential for diagnosing partition-related performance problems.
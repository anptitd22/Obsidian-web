---
title: Adaptive Query Execution (AQE)
source: https://medium.com/@geekycodes/adaptive-query-execution-aqe-in-apache-spark-how-spark-optimizes-your-query-at-runtime-0803cf382012
author:
published:
created: 2026-10-29
description: Spark traditionally builds an execution plan before the query runs.
tags:
  - Spark
---
But here's the problem:

> ***Spark doesn't always know what your data actually looks like until the query starts executing.***

Statistics may be incomplete. Data may be smaller than expected. Partitions may be highly uneven. A join that looked expensive during planning may turn out to be much cheaper at runtime.

This is where **Adaptive Query Execution (AQE)** comes in.

AQE allows Spark SQL to use **runtime statistics to adapt parts of the physical execution plan while the query is running**.

Instead of:

```
SQL Query

   ↓

Logical Plan

   ↓

Physical Plan

   ↓

Execute

   ↓

Done
```

AQE introduces feedback:

```
SQL Query

   ↓

Initial Physical Plan

   ↓

Execute

   ↓

Runtime Statistics

   ↓

AQE Re-optimizes

   ↓

Updated Physical Plan

   ↓

Continue Execution
```

This makes AQE one of the most important concepts for understanding **modern Spark performance optimization**.

In this article, we'll understand:

- What AQE is
- Why static query optimization has limitations
- How AQE works
- Post-shuffle partition coalescing
- Skew join optimization
- Dynamic join strategy changes
- AQE configuration
- PySpark examples
- How to inspect AQE
- Common mistakes
- Production troubleshooting
- AQE interview questions

### 1\. What Is Adaptive Query Execution?

**Adaptive Query Execution**, commonly called **AQE**, is a Spark SQL optimization mechanism that uses **runtime statistics to adjust the physical execution plan**.

The key word is:

> ***Runtime***

Traditional optimization happens primarily before execution.

AQE can make decisions after Spark has observed actual data statistics.

For example, Spark initially expects:

```
Shuffle output

= 1 TB
```

But after execution it discovers:

```
Actual shuffle output

= 20 GB
```

That information can influence subsequent execution decisions.

Conceptually:

```
Initial Plan

                     |

                     ↓

              Execute Stage

                     |

                     ↓

            Runtime Statistics

                     |

          ┌──────────┼──────────┐

          ↓          ↓          ↓

       Partition   Join       Skew

        Sizes     Strategy   Detection

          |          |          |

          └──────────┼──────────┘

                     ↓

                 AQE

                     ↓

            Revised Execution
```

### 2\. Why Do We Need AQE?

Let's consider a simple join:

```
result = orders.join(

    customers,

    "customer_id"

)
```

Before execution, Spark may have statistics suggesting:

```
Orders

= 500 GB

Customers

= 20 GB
```

The optimizer builds a physical plan based on available information.

But after filtering:

```
orders = orders.filter(

    orders.status == "COMPLETED"

)
```

the actual dataset might be much smaller:

```
Orders

500 GB

   ↓

Filter

   ↓

15 GB
```

The runtime reality is now very different from the original estimate.

AQE can use runtime information to make better decisions for parts of the remaining execution.

This gives us the central idea:

> ***AQE lets Spark optimize with knowledge it only obtains during execution.***

### 3\. Static Optimization vs Adaptive Optimization

Without AQE, think of the process as:

```
Query

  ↓

Analyze

  ↓

Optimize

  ↓

Physical Plan

  ↓

Execute
```

With AQE:

```
Query

  ↓

Initial Plan

  ↓

Execute

  ↓

Observe Runtime Statistics

  ↓

Adapt

  ↓

Execute Optimized Plan

  ↓

Adapt Again When Applicable
```

The second approach is more dynamic.

### 4\. AQE Is Not a Replacement for Catalyst

This is an important distinction.

Spark SQL already has the **Catalyst optimizer**.

Catalyst performs query optimization before and during query planning.

AQE builds on Spark SQL's execution framework by using runtime information to adapt execution.

Think of the relationship as:

```
Spark SQL

                    |

          ┌─────────┴─────────┐

          ↓                   ↓

      Catalyst               AQE

   Compile-time /         Runtime

   plan optimization      adaptation

          |                   |

          └─────────┬─────────┘

                    ↓

             Efficient Execution
```

So don't say:

> *"AQE replaces Catalyst."*

A better explanation is:

> ***Catalyst optimizes the query plan, while AQE can adapt execution using runtime statistics.***

### 5\. AQE's Major Optimizations

AQE is particularly important because it can address several practical Spark performance problems.

The major capabilities to know are:

### 1\. Post-shuffle partition coalescing

Combines small shuffle partitions.

### 2\. Skew join optimization

Detects and handles heavily skewed join partitions.

### 3\. Dynamic join strategy changes

Can change certain join strategies based on runtime statistics.

These three areas are extremely important for Spark interviews.

### 6\. Optimization #1 — Coalescing Post-Shuffle Partitions

Let's start with a common problem.

Suppose:

```
spark.sql.shuffle.partitions = 2000
```

Spark creates many shuffle partitions.

But the actual data is much smaller than expected.

You might end up with:

```
2000 partitions
```

where many contain only a small amount of data.

For example:

```
Partition 1 → 5 MB

Partition 2 → 2 MB

Partition 3 → 7 MB

Partition 4 → 1 MB

...

Partition 2000 → 3 MB
```

Running thousands of tiny tasks creates overhead.

AQE can dynamically **coalesce post-shuffle partitions**.

Conceptually:

```
Before AQE

2000

small partitions

P1 P2 P3 P4 P5 P6 ...

 ↓  ↓  ↓  ↓  ↓  ↓

        AQE

          ↓

Fewer larger partitions

P1+P2+P3

P4+P5

P6+P7+P8

...
```

The objective is to reduce unnecessary task overhead while maintaining useful parallelism.

### 7\. Why Post-Shuffle Coalescing Matters

Suppose a query produces:

```
1 GB
```

after a shuffle.

But Spark initially creates:

```
2000 partitions
```

That's only around:

```
~0.5 MB per partition
```

on average.

Creating and scheduling 2000 tasks for such a workload may be inefficient.

AQE can observe the actual shuffle partition sizes and combine them.

Conceptually:

```
2000 tiny tasks

       ↓

AQE

       ↓

100 useful tasks
```

The exact result depends on runtime statistics and configuration.

### 8\. AQE Configuration for Coalescing

AQE is controlled by:

```
spark.conf.get(

    "spark.sql.adaptive.enabled"

)
```

To check whether AQE is enabled:

```
print(

    spark.conf.get(

        "spark.sql.adaptive.enabled"

    )

)
```

You can also inspect the coalescing setting:

```
print(

    spark.conf.get(

        "spark.sql.adaptive.coalescePartitions.enabled"

    )

)
```

A typical configuration is:

```
spark.conf.set(

    "spark.sql.adaptive.enabled",

    "true"

)

spark.conf.set(

    "spark.sql.adaptive.coalescePartitions.enabled",

    "true"

)
```

The exact defaults can depend on Spark version and distribution, so production environments should be checked rather than assuming a particular default.

### 9\. Optimization #2 — Skew Join Optimization

Now consider a different problem.

Suppose we have:

```
Partition 1 → 10 GB

Partition 2 → 11 GB

Partition 3 → 9 GB

Partition 4 → 500 GB
```

The fourth partition is severely skewed.

A join involving that partition can create a very slow task.

Without handling skew:

```
Task 1 → 2 minutes

Task 2 → 2 minutes

Task 3 → 2 minutes

Task 4 → 45 minutes
```

AQE can detect skewed shuffle partitions and, when its skew-join optimization applies, split skewed partitions into smaller pieces.

Conceptually:

```
Before

Partition 4

500 GB

        ↓

       AQE

        ↓

Partition 4A → 100 GB

Partition 4B → 100 GB

Partition 4C → 100 GB

Partition 4D → 100 GB

Partition 4E → 100 GB
```

Now those pieces can be processed more concurrently.

### 10\. Why AQE Skew Handling Is Powerful

Previously, developers often had to manually implement techniques such as **salting** to address severe skew.

Salting is still useful in some workloads.

But AQE can automatically identify certain skewed shuffle partitions at runtime.

This gives Spark an important advantage:

```
Static knowledge

      ↓

Not enough information

      ↓

Execute

      ↓

Detect actual skew

      ↓

AQE

      ↓

Adapt execution
```

This is why AQE should always be considered before manually implementing a complex skew workaround.

### 11\. AQE Skew Join Configuration

You can inspect whether skew join optimization is enabled:

```
print(

    spark.conf.get(

        "spark.sql.adaptive.skewJoin.enabled"

    )

)
```

Relevant AQE settings include:

```
spark.sql.adaptive.skewJoin.enabled

spark.sql.adaptive.skewJoin.skewedPartitionFactor

spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes
```

The factor and byte threshold work together to determine when a partition can be considered skewed.

Don't blindly change these values.

First understand:

- Partition sizes
- Median partition size
- Shuffle volume
- Cluster memory
- Join behavior
- Actual Spark UI metrics

### 12\. Optimization #3 — Dynamic Join Strategy

One of the most interesting AQE capabilities is that Spark can sometimes change a join strategy using runtime statistics.

Suppose the initial plan doesn't choose a broadcast join because Spark doesn't know enough about the data.

After an earlier stage executes, Spark discovers:

```
Actual table size = 50 MB
```

That may make a broadcast strategy attractive.

Conceptually:

```
Initial estimate

      ↓

Shuffle Join

      ↓

Execute earlier stage

      ↓

Actual size discovered

      ↓

AQE

      ↓

Broadcast Join
```

This is an important reason runtime statistics matter.

### 13\. Local Shuffle Reader

AQE can also use a **local shuffle reader** in certain situations.

Suppose AQE changes the join strategy and a shuffle output no longer needs to be read according to its original partitioning requirements.

Spark may be able to read shuffle data locally rather than enforcing the original distribution.

Conceptually:

```
Traditional

Shuffle Data

     ↓

Network Read

     ↓

Executor

AQE

     ↓

Local Shuffle Reader

     ↓

Use locally available data when possible
```

This can reduce unnecessary network traffic.

This is a more advanced AQE feature, but it is worth knowing for senior-level Spark interviews.

### 14\. A Complete AQE Mental Model

Think of AQE as a feedback loop:

```
QUERY

                |

                ↓

        Initial Physical Plan

                |

                ↓

          Execute Stage

                |

                ↓

       Runtime Statistics

                |

       ┌────────┼────────┐

       ↓        ↓        ↓

   Partition   Skew     Join

    Sizes     Detection Strategy

       |        |        |

       └────────┼────────┘

                ↓

              AQE

                |

                ↓

       Adapt Physical Plan

                |

                ↓

        Execute Remaining

                |

                ↓

             Results
```

This is the most important mental model to remember.

### 15\. Example: AQE With an Aggregation

Consider:

```
result = orders.groupBy(

    "customer_id"

).sum("amount")
```

This can involve a shuffle.

Suppose we configure:

```
spark.conf.set(

    "spark.sql.shuffle.partitions",

    1000

)
```

Spark may initially plan around:

```
1000 shuffle partitions
```

But after the shuffle, AQE sees the actual partition sizes.

If many are small:

```
AQE

 ↓

Coalesce partitions

 ↓

Fewer downstream tasks
```

This can reduce task scheduling overhead.

### 16\. Example: AQE With a Skewed Join

Suppose:

```
orders.join(

    customers,

    "customer_id"

)
```

and one customer has a huge number of transactions.

The shuffle stage produces:

```
Partition 1 → 100 MB

Partition 2 → 120 MB

Partition 3 → 90 MB

Partition 4 → 50 GB
```

AQE can detect that the fourth partition is unusually large.

If the skew optimization criteria are met:

```
50 GB partition

       ↓

AQE

       ↓

Split into smaller pieces

       ↓

Parallel processing
```

This can significantly reduce the impact of the skewed key.

### 17\. AQE vs Salting

You just learned about salting.

So which should you use?

The answer depends on the workload.

### AQE

```
Runtime detection

       ↓

Automatic adaptation
```

### Salting

```
Developer explicitly

changes join keys

       ↓

Manual skew handling
```

A practical approach is:

```
Detect skew

     ↓

Check AQE

     ↓

Enable / tune AQE where appropriate

     ↓

Measure

     ↓

Consider salting if skew remains problematic
```

AQE should not be treated as a guarantee that every skew problem disappears.

### 18\. AQE vs repartition()

These concepts solve different problems.

### repartition()

You explicitly change partitioning:

```
df = df.repartition(

    200,

    "customer_id"

)
```

You are telling Spark:

> *Redistribute this data.*

### AQE

Spark observes runtime information and can adapt parts of execution.

```
Initial plan

     ↓

Runtime statistics

     ↓

Adaptive decision
```

Therefore:

> ***Repartition is an explicit developer decision; AQE is runtime plan adaptation.***

They can work together.

### 19\. AQE vs coalesce()

`coalesce()` is also an explicit operation:

```
df = df.coalesce(50)
```

You're telling Spark to reduce partitions.

AQE can dynamically coalesce **post-shuffle partitions** based on runtime statistics.

So:

```
coalesce()

     ↓

Explicit partition reduction
```

while:

```
AQE

     ↓

Runtime-driven partition coalescing
```

This distinction is important.

### 20\. How to Enable AQE

In modern Spark environments, AQE is commonly enabled by default, but always verify your environment.

Check:

```
spark.conf.get(

    "spark.sql.adaptive.enabled"

)
```

If needed:

```
spark.conf.set(

    "spark.sql.adaptive.enabled",

    "true"

)
```

Then verify:

```
print(

    spark.conf.get(

        "spark.sql.adaptive.enabled"

    )

)
```

You can also inspect:

```
spark.conf.get(

    "spark.sql.adaptive.coalescePartitions.enabled"

)
```

and:

```
spark.conf.get(

    "spark.sql.adaptive.skewJoin.enabled"

)
```

[Apache Spark SQL Performance Tuning](https://spark.apache.org/docs/latest/sql-performance-tuning.html?utm_source=chatgpt.com)

### 22\. How to Inspect AQE in the Physical Plan

Use:

```
df.explain("formatted")
```

For a query involving AQE, Spark may show an adaptive execution plan.

You may see concepts such as:

```
AdaptiveSparkPlan
```

For example:

```
== Physical Plan ==

AdaptiveSparkPlan

+- HashAggregate

   +- Exchange

      +- ...
```

The exact physical plan depends on:

- Spark version
- Query
- Statistics
- Configuration
- Runtime data

Therefore, don't memorize one exact plan.

Understand the concepts.

### 23\. Spark UI and AQE

The Spark UI is especially useful when investigating AQE.

Look at:

### SQL tab

Useful for understanding:

- Query execution
- Physical plan
- Runtime statistics
- SQL operators

### Stages tab

Useful for:

- Task duration
- Shuffle read
- Shuffle write
- Input/output
- Spill

### Tasks

Look for:

```
Most tasks → fast

Few tasks → extremely slow
```

which can indicate skew.

Or:

```
Thousands of tiny tasks
```

which may indicate inefficient partition sizing.

### 24\. A Production Debugging Example

Imagine a Spark job suddenly becomes slow.

You inspect the stage:

```
Task 1 → 20 sec

Task 2 → 22 sec

Task 3 → 19 sec

Task 4 → 21 sec

Task 5 → 45 min
```

First suspicion:

> *Data skew.*

Then inspect the shuffle:

```
Partition 1 → 100 MB

Partition 2 → 120 MB

Partition 3 → 90 MB

Partition 4 → 70 GB
```

Now inspect AQE:

```
spark.conf.get(

    "spark.sql.adaptive.enabled"

)
```

If AQE is disabled, enabling it may be worth testing.

If AQE is already enabled, investigate:

- Skew thresholds
- Join type
- Actual key distribution
- Whether the skew qualifies for AQE handling
- Whether salting is still required

This is much better than immediately increasing cluster size.

### 25\. Why AQE Can Reduce Cloud Costs

Suppose a job initially creates:

```
2000 tiny tasks
```

AQE may reduce unnecessary downstream tasks through post-shuffle coalescing.

Or suppose one skewed partition causes:

```
1 task → 40 minutes
```

and AQE successfully splits the skewed work.

The result can be:

```
Shorter execution

     ↓

Less cluster runtime

     ↓

Potentially lower compute cost
```

This matters particularly for:

- Databricks
- EMR
- Managed Spark clusters
- Large ETL pipelines
- Recurring batch workloads

AQE therefore isn't just a theoretical optimization.

It can have direct operational and cost implications.

### 26\. AQE Does Not Mean "Spark Optimizes Everything"

This is an important misconception.

Enabling:

```
spark.sql.adaptive.enabled = true
```

doesn't mean:

> *"I don't need to optimize my Spark jobs anymore."*

AQE cannot fix every problem.

For example, you may still have:

- Bad file layouts
- Excessive small files
- Poor partitioning
- Inefficient UDFs
- Huge unnecessary scans
- Poor data modeling
- Extremely skewed data
- Driver-side bottlenecks
- Expensive Python processing
- Excessive `collect()`

AQE is one component of Spark optimization.

### 27\. Common AQE Mistakes

### Mistake 1: Assuming AQE is always enabled

Check:

```
spark.conf.get(

    "spark.sql.adaptive.enabled"

)
```

### Mistake 2: Changing AQE settings blindly

Don't change thresholds without understanding the workload.

### Mistake 3: Assuming AQE solves every skew problem

Some skew requires additional techniques.

### Mistake 4: Ignoring Spark UI

AQE is runtime-driven.

You need runtime metrics to understand its impact.

### Mistake 5: Assuming fewer partitions always means faster

Coalescing too aggressively can reduce parallelism.

### Mistake 6: Increasing cluster size instead of fixing the execution plan

More hardware isn't always the answer.

First determine whether the bottleneck is:

```
Shuffle

Skew

Partitioning

Join

Memory

CPU

I/O
```

### 28\. AQE Interview Question

### What is AQE?

A strong answer:

> *"Adaptive Query Execution is a Spark SQL optimization mechanism that uses runtime statistics to modify parts of the physical execution plan while a query is executing. It can coalesce small post-shuffle partitions, optimize certain skewed joins, and in some situations change join strategies based on actual runtime data."*

### 29\. AQE Interview Question

### Why is AQE useful if Catalyst already optimizes queries?

Answer:

> *"Catalyst optimizes the query plan using the information available during planning, but some important information isn't known accurately until execution. AQE uses runtime statistics, such as actual shuffle partition sizes and data sizes, to adapt the physical plan. So AQE complements Catalyst rather than replacing it."*

### 30\. AQE Interview Question

### How does AQE solve data skew?

Answer:

> *"AQE can detect skewed shuffle partitions during execution. For supported skewed joins, Spark can split oversized partitions into smaller pieces and replicate the corresponding data from the other side as needed, allowing the work to be processed more evenly. The exact behavior depends on Spark's AQE skew configuration and the runtime statistics."*

### 31\. AQE Interview Question

### What is post-shuffle partition coalescing?

Answer:

> *"Spark may initially create many shuffle partitions based on the configured shuffle partition count. AQE examines their actual runtime sizes and can combine small post-shuffle partitions into fewer, larger partitions. This reduces unnecessary task scheduling overhead while maintaining useful parallelism."*

### 32\. AQE Interview Question

### Can AQE change a join strategy?

Yes, in supported scenarios.

For example:

```
Initial plan

     ↓

Shuffle-based join

     ↓

Runtime statistics

     ↓

AQE

     ↓

Potentially broadcast-based strategy
```

The important phrase is:

> ***based on runtime statistics and applicable optimizer rules.***

Don't claim AQE will always convert any join into a broadcast join.

### 33\. AQE vs Salting — Interview Answer

If asked:

> ***"If AQE handles skew, why would you ever use salting?"***

A good answer is:

> *"AQE should be the first thing I evaluate because it can automatically handle certain skewed joins at runtime. However, AQE doesn't solve every skew pattern or workload. If a persistent hot-key problem remains and the workload requires more explicit control, targeted salting can be considered. Salting changes the data's effective join keys, while AQE adapts the physical execution based on runtime statistics."*

### 34\. Production Optimization Framework

When a Spark job is slow, use this framework:

```
SLOW SPARK JOB

                       |

                       ↓

                Spark UI / SQL

                       |

          ┌────────────┼────────────┐

          ↓            ↓            ↓

       Shuffle        Skew       Memory

          |            |            |

          ↓            ↓            ↓

    Partitioning      AQE        Executors

          |            |

          ↓            ↓

   repartition?    skew handling?

   coalesce?       salting?

   broadcast?

          |

          └────────────┬────────────┘

                       ↓

                  Benchmark

                       ↓

                Keep / Revert
```

The important principle:

> ***Diagnose first. Optimize second.***

### 35\. Final Mental Model

Think of AQE as Spark's **runtime feedback loop**.

```
QUERY

                  |

                  ↓

           Initial Plan

                  |

                  ↓

             Execute

                  |

                  ↓

       ┌────────────────────┐

       │ Runtime Statistics │

       └─────────┬──────────┘

                 ↓

                AQE

                 |

       ┌─────────┼─────────┐

       ↓         ↓         ↓

   Coalesce    Handle    Change

  Partitions    Skew     Strategy

       |         |         |

       └─────────┼─────────┘

                 ↓

          Better Execution
```

The simplest definition to remember is:

> ***AQE allows Spark to use what it learns during execution to improve the remaining execution plan.***

### Key Takeaways

- AQE stands for **Adaptive Query Execution**.
- It uses runtime statistics to adapt Spark SQL execution.
- AQE complements Catalyst rather than replacing it.
- It can coalesce small post-shuffle partitions.
- It can optimize certain skewed joins.
- It can change certain join strategies based on runtime information.
- Local shuffle reading is another AQE optimization.
- AQE is especially useful when compile-time statistics are incomplete or inaccurate.
- Always verify AQE configuration in your environment.
- Use `explain("formatted")` and Spark UI to understand its behavior.
- AQE doesn't eliminate the need for good partitioning and query design.
- Salting can still be useful when AQE doesn't sufficiently address a skew problem.
- Runtime measurement is essential when evaluating AQE optimizations.

### Continue Learning Apache Spark

You now understand one of the most important features in modern Spark performance tuning: **AQE**.

The next step is to go deeper into the problem AQE frequently helps address:

### Data Skew

We'll examine:

- What creates skew
- How to detect hot keys
- How skew appears in Spark UI
- Skewed joins
- AQE skew handling
- Salting
- Broadcast strategies
- Production debugging patterns

Then we'll combine everything into **Spark Join Optimization**.

### You Might Need This Next

### One Spark task is taking 20× longer than the others?

Don't immediately add more executors.

Check:

```
→ Partition sizes → Shuffle read → Shuffle write → Hot keys → AQE settings → Join strategy → Spill → Executor memory
```

A slow task is often a **data distribution problem**, not simply a lack-of-compute problem.

You've now covered:

```
Partitioning

     ↓

Repartition vs Coalesce

     ↓

Broadcast Join

     ↓

Salting

     ↓

AQE
```


---
title: Spark UI
source: https://medium.com/@geekycodes/spark-ui-how-to-debug-and-optimize-apache-spark-jobs-bcbb8da28103
author:
published:
created: 2026-10-29
description: "A practical guide to Spark’s explain(): logical plans, physical operators, join strategies and AQE, with real plans and a join that got…"
tags:
  - Spark
---
![Post cover image](https://freedium-mirror.cfd/img/medium/700/1*Rk3Ztz1TTKym1QvqDjlXVg.png)

### A practical guide to Spark's explain(): logical plans, physical operators, join strategies and AQE, with real plans and a join that got 2.9× faster

You write ten lines of PySpark. It reads two Parquet tables, joins them, filters, and sums revenue by country. It looks harmless.

```python
from pyspark.sql import SparkSession

from pyspark.sql.functions import sum

spark = SparkSession.builder.appName("SalesAnalytics").getOrCreate()

orders = spark.read.parquet("/data/orders")

customers = spark.read.parquet("/data/customers")

result = (

    orders

    .filter("amount > 1000")

    .join(customers, "customer_id")

    .groupBy("country")

    .agg(sum("amount").alias("total_revenue"))

)

result.show()
```

Then it takes ten minutes instead of one. What's going on?

The code can't tell you. It doesn't say whether Spark filters before or after reading the files, which join algorithm it picked, whether the join shuffles data across the cluster, or whether it's reading columns nobody uses.

**The execution plan can.** And it's one method call away:

```
result.explain()
```

This post walks through how to read that output, which operators matter, and how to use the plan to make changes that actually help. Every plan below is **real output from PySpark 4.2**, run on 12 million orders and 400,000 customers, not a hand-drawn approximation. Only file paths were shortened and the long `FileScan` lines trimmed.

*This is part 32 of the Apache Spark series on GeekyCodes. Part 31 covered Spark UI, which shows what a job did after it ran. This part is about seeing what Spark will do before it runs.*

### 1\. What an explain plan actually is

Spark is lazy. When you write `filter` or `join`, nothing runs. Spark records your intent. When an action like `show()` or `write()` arrives, Spark hands that intent to its query optimizer, **Catalyst**, which turns it into a series of increasingly concrete plans.

![None](https://freedium-mirror.cfd/img/medium/4000/1*z_1_77LRWGspEvRgj2yqdQ.png)

Every DataFrame or SQL query goes through these phases. Different explain() modes show different parts of the journey.

- **Parsed logical plan:** the structure of your query, with names not yet checked.
- **Analyzed logical plan:** column names and types resolved against the schema. A misspelled column fails here.
- **Optimized logical plan:** Catalyst rewrites the query by pushing filters down, dropping unused columns and simplifying expressions.
- **Physical plan:** the concrete strategy, such as which join algorithm, where to shuffle and how to aggregate.

The simplest way to keep these apart:

> **A logical plan says what to compute. A physical plan says how.**

Performance problems almost always live in the physical plan, so that's where most of your attention goes.

One caveat before we start. `explain()` doesn't run anything. It shows Spark's plan based on what it knows before execution. With **Adaptive Query Execution (AQE)** enabled, which is the default since Spark 3.2, Spark can revise parts of that plan at runtime. We'll see that happen later.

### 2\. Five ways to ask Spark for its plan

```
df.explain()                    # physical plan only

df.explain(mode="extended")     # all logical plans + physical plan

df.explain(mode="formatted")    # tree overview + numbered node details

df.explain(mode="cost")         # logical plan with size statistics

df.explain(mode="codegen")      # generated Java code
```

![None](https://freedium-mirror.cfd/img/medium/4000/1*AunMKuuCJs-zwBhtA9PNhg.png)

Rule of thumb: start with formatted, switch to extended to see what the optimizer changed, and use cost to ask why a join wasn't broadcast.

If you live in SQL, the same plans are one keyword away: `EXPLAIN`, `EXPLAIN EXTENDED`, `EXPLAIN FORMATTED`, `EXPLAIN COST`.

### 3\. Watching Catalyst rewrite your query

Here is a tiny query: keep rows where `id > 100`, then select `id`.

```
df = spark.range(1_000_000)

df.filter("id > 100").select("id").explain(mode="extended")
```

Real output:

```
== Parsed Logical Plan ==

'Project ['id]

+- Filter (id#64L > cast(100 as bigint))

   +- Range (0, 1000000, step=1, splits=Some(4))

== Analyzed Logical Plan ==

id: bigint

Project [id#64L]

+- Filter (id#64L > cast(100 as bigint))

   +- Range (0, 1000000, step=1, splits=Some(4))

== Optimized Logical Plan ==

Filter (id#64L > 100)

+- Range (0, 1000000, step=1, splits=Some(4))

== Physical Plan ==

*(1) Filter (id#64L > 100)

+- *(1) Range (0, 1000000, step=1, splits=4)
```

Three things happened between the phases:

1. **Parsed → analyzed:**`'id` lost its leading quote and became `id#64L`, a resolved column with a unique ID. The `L` means it's a `long`.
2. **Constant folding:**`cast(100 as bigint)` became plain `100`.
3. The `Project` vanished. Spark realized that selecting `id` from a source that only has `id` does nothing, so it deleted the step.

That last one matters more than it looks. Catalyst routinely removes, reorders and simplifies the steps you wrote. **Your code is a request, not a recipe.** We'll come back to this when we try to "optimize" a query by hand.

The `*(1)` prefix marks **whole-stage code generation**. Operators that share a number are compiled into a single Java function, which is much faster than calling them one row at a time.

### 4\. Reading a physical plan: start at the bottom

Physical plans are printed upside down compared with how data flows. The **data sources are at the bottom** and the **result is at the top**. Read them from the bottom up.

Here's a real aggregation plan:

```
orders.filter("amount > 1000") \

      .groupBy("customer_id") \

      .agg(sum("amount").alias("total_amount")) \

      .explain()

== Physical Plan ==

AdaptiveSparkPlan isFinalPlan=false

+- HashAggregate(keys=[customer_id#34L], functions=[sum(amount#35)])

   +- Exchange hashpartitioning(customer_id#34L, 200), ENSURE_REQUIREMENTS

      +- HashAggregate(keys=[customer_id#34L], functions=[partial_sum(amount#35)])

         +- Filter (isnotnull(amount#35) AND (amount#35 > 1000.0))

            +- FileScan parquet [customer_id#34L,amount#35] ...

                 PushedFilters: [IsNotNull(amount), GreaterThan(amount,1000.0)],

                 ReadSchema: struct<customer_id:bigint,amount:double>
```

![None](https://freedium-mirror.cfd/img/medium/4000/1*xQrcsS9ywZQbImvZio4A1Q.png)

The same plan drawn as a tree. The Exchange splits the work into two stages, the same stages you'd see in Spark UI.

Reading it bottom-up:

- **FileScan** reads only `customer_id` and `amount`. That's column pruning: the orders table has five columns, but Spark reads two. The filter is also listed under `PushedFilters`, so the Parquet reader can skip row groups whose min/max statistics rule them out.
- **Filter** keeps rows with `amount > 1000`. Spark added `isnotnull(amount)` on its own, because a null can never be greater than 1000.
- **HashAggregate partial\_sum** pre-aggregates inside each partition before anything moves. Instead of shuffling millions of order rows, Spark shuffles one partial sum per customer per partition.
- **Exchange hashpartitioning(customer\_id, 200)** is the shuffle. All rows for a given customer are sent to the same one of 200 partitions.
- **HashAggregate sum** combines the partial sums into the final answer.

If you only remember one operator, make it `Exchange`. It means data moves between executors, usually over the network and through disk. Every `Exchange` is also a **stage boundary**.

Each operator comes with a question worth asking when you see it.

![None](https://freedium-mirror.cfd/img/medium/4000/1*bbVHcCg0l8Z9spGyBlJ-Pw.png)

A quick note on **Sort**. A global `orderBy` shows up like this in a real plan:

```
Sort [amount#35 ASC NULLS FIRST], true, 0

+- Exchange rangepartitioning(amount#35 ASC NULLS FIRST, 200), ENSURE_REQUIREMENTS

   +- FileScan parquet [order_id, customer_id, amount, order_date, status] ...
```

Notice `rangepartitioning` instead of `hashpartitioning`. To sort globally, Spark must first shuffle rows into ordered ranges. Sorting a big table is never free, so check whether you really need it.

None of these operators is bad on its own. A plan with an `Exchange` isn't broken, because a `groupBy` on a new key *needs* a shuffle. The goal is to make sure each one is there for a reason.

### 5\. Joins: where the plan matters most

The join operator in your plan tells you how much data will travel across the cluster. Two strategies dominate.

![None](https://freedium-mirror.cfd/img/medium/4000/1*KrHJEiYcWjQtH5iKrKXiTg.png)

Sort-merge join reshuffles both tables. Broadcast hash join ships the small table everywhere and leaves the big one alone.

### SortMergeJoin

Here's the real physical plan for joining our orders and customers with default settings:

```
+- SortMergeJoin [customer_id#34L], [customer_id#38L], Inner

   :- Sort [customer_id#34L ASC NULLS FIRST], false, 0

   :  +- Exchange hashpartitioning(customer_id#34L, 200)

   :     +- Filter ((isnotnull(amount#35) AND (amount#35 > 1000.0)) AND isnotnull(customer_id#34L))

   :        +- FileScan parquet [customer_id#34L,amount#35] ...

   +- Sort [customer_id#38L ASC NULLS FIRST], false, 0

      +- Exchange hashpartitioning(customer_id#38L, 200)

         +- Filter isnotnull(customer_id#38L)

            +- FileScan parquet [customer_id#38L,country#39] ...
```

The signature is easy to spot: **two Exchanges and two Sorts**, one of each per side. Both tables are shuffled by `customer_id` so matching keys land together, sorted, and then merged. This strategy works for any table size, but you pay for a full shuffle of both inputs.

The:- marks the first child, the left side of the join. Lines starting with: belong to that branch.

### BroadcastHashJoin

If one side is small, Spark can send a full copy of it to every executor instead:

```
from pyspark.sql.functions import broadcast

orders.join(broadcast(customers), "customer_id").explain()

+- BroadcastHashJoin [customer_id#34L], [customer_id#38L], Inner, BuildRight, false, false

   :- Filter ((isnotnull(amount#35) AND (amount#35 > 1000.0)) AND isnotnull(customer_id#34L))

   :  +- FileScan parquet [customer_id#34L,amount#35] ...

   +- BroadcastExchange HashedRelationBroadcastMode(List(input[0, bigint, false]),false)

      +- Filter isnotnull(customer_id#38L)

         +- FileScan parquet [customer_id#38L,country#39] ...
```

Both sorts are gone, and so is the `Exchange` on the orders side. The big table is never shuffled for the join. `BuildRight` tells you the right side, customers, became the in-memory hash table.

The catch is that a broadcast copy must fit in executor memory. Broadcasting something too large can cause out-of-memory errors. **Don't force broadcasts based on row count alone.** Check the actual size.

### 6\. An experiment: does hand-optimizing the code help?

Now let's put this to work. Here's a classic "before and after" that many Spark tutorials recommend, including the full version of this one on GeekyCodes: filter early and select only the columns you need.

The customers table here is deliberately wide: **26 columns**, of which we need just two.

```python
# Version 1: written naively, filter after the join

v1 = (

    orders

    .join(customers, "customer_id")

    .filter("amount > 1000")

    .select("customer_id", "country", "amount")

)

# Version 2: hand-optimized, filter and prune before the join

customers_small = customers.select("customer_id", "country")

v2 = (

    orders

    .filter("amount > 1000")

    .select("customer_id", "amount")

    .join(customers_small, "customer_id")

    .select("customer_id", "country", "amount")

)

# Version 3: hand-optimized + broadcast the small side

v3 = (

    orders

    .filter("amount > 1000")

    .select("customer_id", "amount")

    .join(broadcast(customers_small), "customer_id")

    .select("customer_id", "country", "amount")

)
```

Version 2 looks clearly better. But before timing anything, compare the plans.

**Version 1 and version 2 produce identical physical plans.** Diff them and, apart from internal ID numbers, they match line for line.

![None](https://freedium-mirror.cfd/img/medium/4000/1*UFT1MZkNtfjWNPP7yahz2g.png)

Catalyst had already done the "optimization" for us: pushed the filter into the scan and pruned 24 of 26 customer columns.

Even though version 1 filters *after* the join in the code, its plan shows `amount > 1000` under `PushedFilters` in the orders scan. Even though version 1 never selects customer columns before the join, its `ReadSchema` lists only `customer_id` and `country`. Catalyst rewrote the naive version into the optimized one.

![None](https://freedium-mirror.cfd/img/medium/4000/1*LrEzPuK13fzHRU11-IakyA.png)

Now let's run all three with default Spark settings and AQE on, timing the median of three runs:

Same plan, same cost. A different plan, a different cost.

- **Version 1:** 5.2 s, 72.8 MB shuffled.
- **Version 2:** 5.6 s, 72.8 MB shuffled. The difference is run-to-run noise; version 1's own runs ranged from 4.8 to 7.0 s.
- **Version 3:** 1.8 s, **0 MB shuffled.** That's **2.9× faster**.

**Rewriting the code changed nothing, because it didn't change the plan. Changing the join strategy changed everything.**

This is the most practical lesson an explain plan teaches: *compare plans before you compare stopwatches.* If two versions have the same plan, no amount of benchmarking will separate them.

### So why didn't Spark broadcast on its own?

Spark broadcasts automatically when it thinks one side is smaller than `spark.sql.autoBroadcastJoinThreshold`, which defaults to 10 MB. The `cost` mode shows what Spark thinks:

```
v1.explain(mode="cost")

+- Project [customer_id#38L, country#39], Statistics(sizeInBytes=42.3 MiB)

   +- Relation [customer_id, country, attr_00 ... attr_23] parquet, Statistics(sizeInBytes=606.3 MiB)
```

Spark started from the size of the whole customers table on disk, 606 MiB, and scaled it down for the two columns we kept. Its estimate was **42.3 MiB**, well over the threshold.

At runtime, AQE measured the real thing. The shuffled customers data was **12.2 MiB** uncompressed. That's much closer, but still just above 10 MiB, so AQE also kept the sort-merge join. Both decisions were reasonable by the rules.

![None](https://freedium-mirror.cfd/img/medium/4000/1*L_Se9BLXelXHO7BCiShBbg.png)

Both the estimate and the real size sit above the threshold, so a hint is the way to get the broadcast.

You, however, know something Spark's rules don't: 12 MiB fits comfortably in executor memory. That's exactly when a `broadcast()` hint is the right tool.

While you're in cost mode, look at the estimated size of the join itself: **2.6 PiB.** Without column statistics, Spark has no idea how many rows will match, so its join estimates can be wildly pessimistic. That's why the `cost` mode is worth knowing. When Spark makes a surprising choice, the estimate is often the reason.

### What column pruning saves, when you let it

Catalyst pruned columns automatically because our final `select` only asked for three. What if the query keeps every column?

```
v0 = orders.join(customers, "customer_id").filter("amount > 1000")   # no select
```

Now the plan's `ReadSchema` lists all 26 customer columns, and every one of them is shuffled. Running both versions with AQE off, so each plan runs exactly as printed:

![None](https://freedium-mirror.cfd/img/medium/4000/1*lSccT-ftd3Sg_VCVPvnHOQ.png)

Asking for columns you don't use: 9.8× more data shuffled and 3.4× slower.

Catalyst can only prune what your query doesn't ask for. A `select("*")` near the end of a pipeline, or writing out a full join result "just in case," quietly turns this optimization off. The plan shows it immediately in `ReadSchema`.

### 7\. AQE: the plan can change after it starts

When AQE is on, the plan starts with `AdaptiveSparkPlan isFinalPlan=false`. That means *this is the starting plan, and it may change.* Run an action, call `explain()` again on the same DataFrame, and you get both versions. Here's an excerpt of the real final plan for our revenue-by-country query:

```
AdaptiveSparkPlan isFinalPlan=true

+- == Final Plan ==

   ResultQueryStage 3

   +- *(6) HashAggregate(keys=[country#39], functions=[sum(amount#35)])

      +- AQEShuffleRead coalesced

         +- ShuffleQueryStage 2

            +- Exchange hashpartitioning(country#39, 200)

               ...

                     +- *(5) SortMergeJoin [customer_id#34L], [customer_id#38L], Inner

                        :- *(3) Sort ...

                        :  +- AQEShuffleRead coalesced

                        :     +- ShuffleQueryStage 0

               ...

+- == Initial Plan ==

   ...
```

What to look for:

- `isFinalPlan=true`: this is what actually ran.
- `ShuffleQueryStage`: the points where AQE paused to measure real data sizes.
- `AQEShuffleRead coalesced`: AQE merged many small shuffle partitions into fewer, larger ones. We only have six countries, so most of the 200 partitions would have been empty.
- **The join strategy**: here it stayed `SortMergeJoin`, for the 12.2 MiB reason above. With a smaller customers side, AQE could switch it to a broadcast join at runtime.

The takeaway is that a pre-execution plan under AQE is a starting point, not the final word. To see what really ran, check the plan after execution, or open the SQL / DataFrame tab in Spark UI.

### 8\. Plan clues, and what to do about them

![None](https://freedium-mirror.cfd/img/medium/4000/1*VXIUdR_dKG7tNipLVVusvg.png)

Use these to form a hypothesis, not a verdict. The plan doesn't show how much data moved or how long anything took.

Two of these deserve extra emphasis:

- `PushedFilters: []` when your code clearly filters. The most common culprit is a Python UDF. Catalyst can't see inside Python functions, so it can't push them into the reader. The second most common is wrapping the column in a cast or expression. Prefer built-in functions whenever you can.
- `SortMergeJoin` where one side is small. Check the estimate with `cost` mode, check the real size in Spark UI, and test `broadcast()` if it's genuinely small.

### 9\. explain() and Spark UI are two halves of one loop

`explain()` tells you what Spark *intends* to do. Spark UI tells you what it *actually cost*. Neither is enough alone. A plan can show an `Exchange` without telling you whether it moved 5 MB or 500 GB, and Spark UI can show you a slow stage without telling you why the optimizer chose it.

![None](https://freedium-mirror.cfd/img/medium/4000/1*SC-7D0FzLfuqvEnCeXfpXw.png)

Plan, hypothesize, run, measure, change one thing, and repeat.

The workflow that works:

1. Write the query and call `explain(mode="formatted")` before running it.
2. Check the scans, count the Exchanges and look at the join operator.
3. Form a hypothesis, such as "the shuffle on customers is the cost."
4. Run it and confirm in Spark UI with shuffle sizes, spill and task times.
5. Change **one** thing, re-explain to confirm the plan changed, then measure again.

Step 5's "re-explain" is the step people skip. Our experiment in Section 6 shows why it matters. If the plan didn't change, the runtime won't either.

### 10\. Seven mistakes to avoid

1. Treating every `Exchange` as a bug. Joins and aggregations on new keys need shuffles. Remove only the unnecessary ones.
2. Assuming `SortMergeJoin` is always slow. For two large tables it's usually the right choice. Broadcasting a big table is worse.
3. **Treating the plan as a runtime report.** It shows operators, not durations or data volumes.
4. **Assuming a rewritten query must be faster.** Catalyst may already have done the rewrite, as we saw. Compare the plans.
5. **Ignoring statistics.** Join choices depend on size estimates, and those estimates can be far off. `explain(mode="cost")` shows them.
6. **Ignoring AQE.** The initial plan isn't necessarily what ran. Check `isFinalPlan`.
7. **Optimizing by operator name alone.** Use the plan to form a hypothesis, then validate it with real metrics.

### Quick interview prep

**What's the difference between a logical and a physical plan?** The logical plan describes what to compute: filters, projections, joins. The physical plan picks how: specific algorithms like `SortMergeJoin` and data movement like `Exchange`.

What does `Exchange` mean? Data redistribution, usually a shuffle. Each one creates a stage boundary.

**How do you check whether filter pushdown is working?** Look at the `FileScan` node. Pushed filters appear under `PushedFilters`, and filters on partition columns appear under `PartitionFilters`.

**How do you recognize a broadcast join?** A `BroadcastHashJoin` with a `BroadcastExchange` under one side. `BuildLeft` or `BuildRight` tells you which side.

**Why might Spark not broadcast a small table?** Its size estimate exceeds `autoBroadcastJoinThreshold`. That happens when statistics are missing or the table is the result of transformations Spark can't estimate well.

**Does the plan tell you how long a query will take?** No. It shows the strategy. Runtime depends on data volume, skew, resources and I/O.

### The takeaway

The explain plan turns Spark from a black box into something you can reason about before spending cluster time. In our experiment it did three things no stopwatch could:

- It showed that a "hand-optimized" rewrite was **identical** to the original, because Catalyst had already done the work.
- It showed **why** Spark chose the slower join: a 42 MiB estimate against a 10 MiB threshold.
- It confirmed that a one-line `broadcast()` hint **removed both shuffles** before we ran anything. The result was 2.9× faster.

So the habit to build is simple: **explain → understand → hypothesize → change one thing → measure.**

**Next in the series: Part 33, Memory Management in Apache Spark.** We'll look at how executors split memory between execution and storage, why spill happens, and how memory pressure slows jobs down.

*Want the full Apache Spark learning path, from fundamentals to performance tuning? It's on* [GeekyCodes](https://geekycodes.in/apache-spark-tutorials/). If this helped, follow for the next part.

**Further reading** — [Spark SQL Performance Tuning](https://spark.apache.org/docs/latest/sql-performance-tuning.html), the official guide to AQE, join hints and broadcast thresholds — [EXPLAIN statement reference](https://spark.apache.org/docs/latest/sql-ref-syntax-qry-explain.html) — [Previous part: Spark UI, How to Debug and Optimize Apache Spark Jobs](https://geekycodes.in/apache-spark-tutorials/) — [Lazy Evaluation in Apache Spark](https://geekycodes.in/data-engineering/lazy-evaluation-in-apache-spark-why-your-code-doesnt-run-when-you-write-it/)

Experiment setup: PySpark 4.2 in local mode with 4 cores. Synthetic Parquet data: 12M orders (5 columns), 400K customers (26 columns), about 6M orders with amount > 1000. Timings are the median of 3 runs, written to Spark's `noop` sink; shuffle sizes come from Spark's monitoring REST API. Absolute times depend on hardware; the ratios are what carry over.
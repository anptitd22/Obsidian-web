---
title: Checkpointing
source: https://medium.com/@geekycodes/checkpointing-in-apache-spark-how-to-break-long-lineage-and-improve-reliability-a5dc51da18d4
author:
published:
created: 2026-10-29
description: Learn how checkpoint() works, why long execution plans become expensive, and when reliable or local checkpointing makes sense in production…
tags:
  - Spark
---
Learn how `checkpoint()` works, why long execution plans become expensive, and when reliable or local checkpointing makes sense in production PySpark applications.

### Introduction: What Happens When Your Spark Lineage Becomes Too Long?

Imagine building a PySpark pipeline that repeatedly transforms a DataFrame over hundreds of iterations.

Each iteration depends on the previous result. As the computation grows, Spark must manage an increasingly complex chain of dependencies.

Conceptually, the pipeline looks like this:

```
Source Data

    |

Transformation 1

    |

Transformation 2

    |

Transformation 3

    |

    ...

    |

Transformation 100

    |

Final Result
```

This dependency chain is called **lineage**.

Lineage is one of Spark's strengths: if a partition is lost, Spark can often recompute it from the original data and transformations.

But extremely long lineages can become problematic. They can increase planning overhead, make iterative workloads harder to manage, and cause expensive recomputation when intermediate results are lost.

This is where checkpointing becomes useful.

```
Before checkpointing

Source → A → B → C → D → E → F → G

After checkpointing

Source → A → B → C

                  |

             CHECKPOINT

                  |

                  D → E → F → G
```

Checkpointing saves an intermediate result and allows Spark to truncate the dependency chain for subsequent computation.

The central idea is simple:

> ***Caching helps reuse computed data. Checkpointing creates a durable boundary in the computation so Spark no longer needs the entire previous lineage to produce the checkpointed result.***

In this tutorial, we'll explore reliable checkpointing, local checkpointing, practical PySpark examples, and the mistakes that matter in production.

If you're finding this series useful, **follow GeekyCodes and subscribe by email** so you don't miss the upcoming tutorials on Spark internals, performance optimization, and production data engineering.

### 1\. What Is Checkpointing in Apache Spark?

Checkpointing is a mechanism for saving a computed dataset to a checkpoint location and truncating its previous dependency chain.

Spark supports checkpointing for both RDDs and DataFrames, although their APIs differ.

For DataFrames, the primary methods are:

```
df.checkpoint()
```

and:

```
df.localCheckpoint()
```

The distinction is important:

- `checkpoint()` uses a configured checkpoint directory and provides a reliable checkpoint.
- `localCheckpoint()` uses the caching subsystem on executors and is not reliable in the same way.

The official [PySpark DataFrame checkpoint documentation](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.checkpoint.html) describes checkpointing as a way to truncate the logical plan, particularly for iterative algorithms whose plans can grow substantially.

### A simple example

```
df = spark.read.parquet("sales/")

filtered_df = df.filter("amount > 1000")

checkpointed_df = filtered_df.checkpoint()
```

Conceptually, Spark materializes the intermediate result into the configured checkpoint directory. The returned DataFrame can then be used without depending on the original transformation chain in the same way.

**Important:** Reliable DataFrame checkpointing is eager by default. Calling `checkpoint()` without arguments triggers the checkpoint computation rather than merely marking the DataFrame for later persistence.

### 2\. Why Does Spark Need Checkpointing?

Consider an iterative algorithm.

Each iteration updates a DataFrame using the previous iteration's result:

```
result = initial_df

for i in range(100):

    result = result.withColumn(

        "score",

        result["score"] * 1.01

    )
```

This is a simplified example of a growing transformation chain.

Conceptually:

```
Iteration 1

    |

Iteration 2

    |

Iteration 3

    |

   ...

    |

Iteration 100
```

In real iterative workloads, transformations can be much more complex, involving joins, filters, aggregations, graph operations, or multiple branches.

As the chain grows, Spark may need to manage a large logical plan.

Checkpointing can establish a boundary:

```
Iterations 1–25

       |

       v

  CHECKPOINT

       |

       v

Iterations 26–50

       |

       v

  CHECKPOINT

       |

       v

Iterations 51–75

       |

       v

  CHECKPOINT

       |

       v

Iterations 76–100
```

Instead of maintaining one ever-growing dependency chain, subsequent computation can start from checkpointed intermediate results.

### When is this useful?

- Iterative algorithms.
- Graph processing.
- Repeated transformations that create increasingly large plans.
- Long-running pipelines where lineage management becomes a problem.
- Workloads where recomputing an extensive upstream dependency chain is expensive.

Checkpointing is not required for every Spark job. It is most useful when the benefits of truncating lineage justify the cost of writing and materializing the intermediate data.

### 3\. Understanding Lineage: The Foundation of Checkpointing

Before looking at code, let's understand the concept checkpointing changes.

Suppose you build a pipeline:

```
df = spark.read.parquet("transactions/")

step1 = df.filter("amount > 0")

step2 = step1.withColumn(

    "amount_with_tax",

    step1["amount"] * 1.18

)

step3 = step2.filter(

    "amount_with_tax > 1000"

)
```

Spark tracks the dependencies required to produce `step3`.

Conceptually:

```
Parquet Source

      |

    Filter

      |

  withColumn

      |

    Filter

      |

  Final Result
```

If an intermediate partition is unavailable, Spark may be able to reconstruct it using the source and the required transformations.

This is useful, but the dependency chain can become expensive to manage when it grows very large.

Checkpointing creates a new starting point:

```
Parquet Source

      |

    Filter

      |

  withColumn

      |

    Filter

      |

  CHECKPOINT

      |

  Checkpointed Data

      |

  New Transformations
```

After reliable checkpointing completes, Spark can use the checkpointed data instead of tracing the entire original lineage to compute subsequent results.

### 💡 Going Deeper

Checkpointing does not mean that Spark stops using execution plans altogether. It changes the dependencies of the checkpointed result. Spark still optimizes and executes subsequent transformations using its normal execution machinery.

### 4\. Types of Checkpointing in Spark

There are two primary checkpointing approaches to understand in PySpark.

Let's examine both.

### 5\. Reliable Checkpointing with checkpoint()

Reliable checkpointing saves the computed result to the configured checkpoint directory.

The location should be accessible to the Spark application and appropriate for your deployment. In a distributed production environment, use a reliable shared filesystem or supported distributed storage location rather than assuming that a path on the driver's local machine is accessible to all workers.

### Step 1: Configure the checkpoint directory

```
spark.sparkContext.setCheckpointDir(

    "s3a://my-bucket/spark-checkpoints/"

)
```

The example assumes that your Spark environment has the necessary S3 filesystem configuration, credentials, and dependencies.

You can also use an appropriate HDFS or other supported checkpoint location.

### Step 2: Read the source data

```
df = spark.read.parquet(

    "s3a://my-bucket/transactions/"

)
```

### Step 3: Apply transformations

```
from pyspark.sql.functions import col

transformed_df = (

    df

    .filter(col("amount") > 0)

    .withColumn(

        "amount_with_tax",

        col("amount") * 1.18

    )

    .filter(col("amount_with_tax") > 1000)

)
```

### Step 4: Checkpoint the result

```
checkpointed_df = transformed_df.checkpoint()
```

By default, `eager=True`, so the checkpoint is computed immediately.

### Step 5: Continue processing

```
final_df = checkpointed_df.groupBy(

    "region"

).sum("amount_with_tax")
```

The resulting computation can use the checkpointed data as its input rather than relying on the entire upstream transformation chain.

### The complete workflow

```
Source Data

     |

Transformations

     |

Checkpoint Computation

     |

Reliable Checkpoint Files

     |

Checkpointed DataFrame

     |

More Transformations

     |

Final Output
```

### ⚡ Performance Tip

Checkpointing introduces real work: Spark must compute the data and write it to storage. Use it when reducing lineage complexity or avoiding expensive recomputation is worth that additional cost.

### 6\. What Does eager=True Mean?

The DataFrame checkpoint API accepts an `eager` parameter.

```
checkpointed_df = df.checkpoint(eager=True)
```

With eager checkpointing, Spark computes and saves the checkpoint as part of the call.

By contrast:

```
checkpointed_df = df.checkpoint(eager=False)
```

requests lazy checkpointing.

With lazy checkpointing, the checkpoint is not immediately materialized. Its execution is deferred until a later action requires the data.

For example:

```
spark.sparkContext.setCheckpointDir(

    "s3a://my-bucket/spark-checkpoints/"

)

checkpointed_df = df.checkpoint(eager=False)

# A later action triggers execution

checkpointed_df.count()
```

For most introductory and production examples, **eager checkpointing is easier to reason about** because the checkpoint boundary is materialized immediately.

If you use lazy checkpointing, verify that the checkpoint has actually been materialized before relying on it as a completed checkpoint.

### 7\. Local Checkpointing with localCheckpoint()

Local checkpointing also truncates lineage, but it uses the executor-side caching subsystem rather than a reliable checkpoint directory.

Example:

```
local_df = transformed_df.localCheckpoint()
```

Like `checkpoint()`, `localCheckpoint()` defaults to `eager=True`.

The advantage is that it can avoid the overhead of writing a reliable checkpoint to distributed storage.

The disadvantage is reliability.

If an executor fails and the local checkpoint data becomes unavailable, Spark may be unable to reconstruct the lost data from the truncated lineage.

### Visual comparison

Reliable checkpoint:

```
Transformations

      |

      v

Checkpoint Files

(reliable storage)

      |

      v

Subsequent Computation
```

Local checkpoint:

```
Transformations

      |

      v

Executor-side Storage

      |

      v

Subsequent Computation
```

The second approach can be faster in suitable workloads, but the storage does not provide the same recovery guarantees.

The official [PySpark localCheckpoint documentation](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.localCheckpoint.html) explicitly warns that local checkpoints are not reliable.

### When should you use local checkpointing?

Consider it when:

- Lineage truncation is important.
- The overhead of reliable checkpoint storage is undesirable.
- The workload can tolerate the increased failure risk.
- Executor lifecycle and storage behavior are understood.

Avoid treating local checkpointing as a drop-in replacement for reliable checkpointing in critical production pipelines.

### 8\. Checkpointing vs Caching vs Persistence

These three concepts are related, but they solve different problems.

Consider the following:

```
df.cache()
```

This requests persistence for reuse.

```
df.persist()
```

This requests persistence and allows explicit storage-level selection.

```
checkpointed_df = df.checkpoint()
```

This creates a checkpointed result and truncates the previous lineage.

### Remember this distinction

```
CACHE / PERSIST

      |

      v

Keep computed data for reuse

CHECKPOINT

      |

      v

Save computed data and truncate lineage
```

Caching and persistence do not, by themselves, replace the original lineage with a reliable checkpoint.

For a deeper explanation, read the previous tutorials on **Caching** and **Persistence in Apache Spark**.

### 9\. A Practical Example: Iterative DataFrame Processing

Let's consider a simplified iterative transformation workload.

```
from pyspark.sql import SparkSession

from pyspark.sql.functions import col

spark = (

    SparkSession.builder

    .appName("CheckpointingExample")

    .getOrCreate()

)

spark.sparkContext.setCheckpointDir(

    "s3a://my-bucket/spark-checkpoints/"

)

df = spark.read.parquet(

    "s3a://my-bucket/transactions/"

)

result = df

for iteration in range(10):

    result = result.withColumn(

        "adjusted_amount",

        col("amount") * (1 + 0.01 * (iteration + 1))

    )

    if (iteration + 1) % 5 == 0:

        result = result.checkpoint()

result.show(10, truncate=False)
```

This example establishes a checkpoint after every five iterations.

Conceptually:

```
Initial DataFrame

       |

Iterations 1–5

       |

   CHECKPOINT

       |

Iterations 6–10

       |

   CHECKPOINT

       |

Final Result
```

**A technical caveat:** This example illustrates periodic lineage truncation, not a useful business calculation. Each iteration replaces `adjusted_amount` using the original `amount` column, so it does not cumulatively update the previous iteration's value. Real iterative algorithms must define their update logic carefully.

Also, checkpointing every five iterations is only an illustrative choice. The right frequency depends on plan growth, computation cost, checkpoint I/O, and failure-recovery requirements.

### 10\. Checkpointing RDDs

Spark also supports checkpointing at the RDD level.

Unlike the DataFrame API, the RDD API uses a different pattern: mark the RDD for checkpointing before the action that materializes it.

```
sc = spark.sparkContext

sc.setCheckpointDir(

    "s3a://my-bucket/spark-checkpoints/"

)

rdd = sc.parallelize(range(1000))

processed_rdd = rdd.map(

    lambda x: x * 2

)

processed_rdd.checkpoint()

# Trigger execution and materialize the checkpoint

processed_rdd.count()

print(processed_rdd.isCheckpointed())

print(processed_rdd.getCheckpointFile())
```

The important details are:

1. Configure the checkpoint directory.
2. Call `checkpoint()` on the RDD before its materializing action.
3. Trigger an action.
4. Inspect the checkpoint status or file location if needed.

The RDD documentation recommends persisting an RDD before checkpointing when appropriate, so Spark can reuse computed partitions instead of recomputing them just to write the checkpoint.

See the official [RDD programming guide](https://spark.apache.org/docs/latest/rdd-programming-guide.html) for details.

### 11\. How Checkpointing Affects the Execution Plan

Suppose a pipeline has many transformations:

```
Source

  |

Filter

  |

Join

  |

Aggregation

  |

Window Function

  |

More Transformations

  |

Final Result
```

The execution and dependency structure can become complicated, particularly in iterative applications.

After checkpointing an intermediate result:

```
Source

  |

Filter

  |

Join

  |

Aggregation

  |

CHECKPOINT

  |

Checkpointed Data

  |

Window Function

  |

More Transformations

  |

Final Result
```

The downstream computation can use the checkpointed data as its starting point.

### 💡 Going Deeper: Logical Plans vs Lineage

For DataFrames, Spark uses structured logical and physical plans. Checkpointing can truncate the logical plan as well as establish a new dependency boundary for the computed result.

These concepts are related, but not identical to the RDD lineage and stage structure used by Spark's execution engine.

Checkpointing does not guarantee that every downstream operation will become faster. It is most valuable when the reduction in plan complexity or recomputation outweighs the cost of materializing the checkpoint.

For more detail, see the official [DataFrame checkpoint API](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.checkpoint.html).

### 12\. Common Checkpointing Mistakes

### Mistake 1: Forgetting to configure the checkpoint directory

Incorrect:

```
checkpointed_df = df.checkpoint()
```

when no checkpoint directory has been configured.

For reliable DataFrame checkpointing, configure the directory first:

```
spark.sparkContext.setCheckpointDir(

    "s3a://my-bucket/spark-checkpoints/"

)

checkpointed_df = df.checkpoint()
```

Make sure the location is accessible and writable.

### Mistake 2: Checkpointing every transformation

Checkpointing has a cost.

If you checkpoint after every operation, the pipeline may spend too much time materializing intermediate results.

Instead, checkpoint at meaningful boundaries where lineage growth or recovery requirements justify it.

### Mistake 3: Assuming local checkpointing is fault-tolerant

```
df.localCheckpoint()
```

is not equivalent to a reliable checkpoint. Executor loss can make local checkpoint data unavailable.

### Mistake 4: Confusing checkpointing with caching

Caching is primarily about reuse. Checkpointing is primarily about lineage truncation.

Choose based on the problem you are trying to solve.

### Mistake 5: Choosing a checkpoint frequency without measurement

A checkpoint interval that works for one pipeline may be wasteful in another.

Measure plan growth, execution time, checkpoint-write overhead, and recovery needs.

### Mistake 6: Using an unsuitable storage location

A checkpoint path that is local to one machine may be inappropriate for a distributed cluster.

Use storage that fits your deployment architecture and reliability requirements.

### 13\. How to Decide Whether Checkpointing Is Necessary

Use this practical decision process:

```
Is the lineage or plan becoming expensive to manage?

                  |

          ┌───────┴────────┐

          |                |

         No               Yes

          |                |

   Avoid unnecessary   Is lineage truncation

    checkpointing       actually needed?

                           |

                    ┌──────┴──────┐

                    |             |

                   No            Yes

                    |             |

               Tune the plan   Choose checkpoint

               or pipeline    strategy

                                  |

                         ┌────────┴────────┐

                         |                 |

                  Reliability needed?   Risk acceptable?

                         |                 |

                     Reliable          Local checkpoint

                     checkpoint
```

Before introducing checkpointing, consider whether a simpler transformation pipeline, fewer unnecessary iterations, or better data layout could solve the problem.

Checkpointing is a useful tool, not a substitute for sound pipeline design.

### 14\. Production Checklist

Before deploying checkpointing in a production Spark job, verify the following:

- The checkpoint directory is configured and writable.
- The storage location is appropriate for a distributed environment.
- The workload has a genuine lineage-management or recovery requirement.
- Checkpointing frequency is based on measurement rather than guesswork.
- The cost of writing checkpoint data is acceptable.
- Local checkpointing is used only when its reliability trade-off is acceptable.
- Checkpoint storage capacity and lifecycle are managed.
- The resulting plan and execution time are validated.
- Failure behavior has been considered.
- The application has been tested under realistic cluster conditions.

### 15\. Common Interview Questions

### 1\. What is checkpointing in Apache Spark?

Checkpointing saves an intermediate result and truncates its previous lineage, reducing dependency-chain complexity and avoiding the need to recompute the entire upstream chain for subsequent operations.

### 2\. Why is checkpointing useful for iterative algorithms?

Iterative algorithms can repeatedly extend their dependency chains. Checkpointing periodically materializes results and prevents lineage or logical-plan growth from becoming unmanageable.

### 3\. What is the difference between checkpoint() and localCheckpoint()?

`checkpoint()` uses a configured checkpoint directory and provides a reliable checkpoint. `localCheckpoint()` uses executor-side caching storage and sacrifices reliability for lower overhead.

### 4\. Is checkpointing an action?

For DataFrames, `checkpoint()` defaults to `eager=True`, which materializes the checkpoint immediately. It can also be called with `eager=False`. For RDDs, `checkpoint()` marks the RDD, and an action is needed to materialize the checkpoint.

### 5\. Does checkpointing improve performance automatically?

No. It adds computation and storage I/O. It can improve workloads where reducing lineage complexity or avoiding expensive recomputation outweighs those costs.

### 6\. Does caching truncate lineage?

No. Caching and persistence retain computed data for reuse, but they do not provide the same lineage-truncation semantics as checkpointing.

### 7\. Where should reliable checkpoints be stored?

In a location suitable for the deployment, typically a reliable shared or distributed filesystem accessible to the Spark application.

### 8\. When would you avoid checkpointing?

For short, straightforward pipelines where lineage is manageable and the checkpoint's materialization cost would outweigh its benefits.

### 9\. Can a local checkpoint be lost?

Yes. Because local checkpoint data is held through executor-side storage, executor loss can make the data unavailable.

### 10\. How would you choose a checkpoint interval?

I would profile the workload, monitor plan growth and recomputation costs, measure checkpoint I/O, and choose a boundary that balances execution overhead against reliability and lineage-management needs.

### Key Takeaways

- **Checkpointing truncates lineage** by saving an intermediate result.
- Reliable checkpointing uses a configured checkpoint directory.
- `DataFrame.checkpoint()` is eager by default.
- `DataFrame.localCheckpoint()` uses executor-side caching storage and is not reliable against loss of that data.
- RDD checkpointing uses a different API pattern and must be materialized by an action.
- Checkpointing is particularly useful for iterative algorithms and long dependency chains.
- Caching and persistence are primarily about reuse; checkpointing is about lineage truncation.
- Checkpointing introduces storage and execution overhead, so use it selectively.
- Measure the results rather than assuming checkpointing automatically makes a job faster.

### The Mental Model to Remember

```
CACHE / PERSIST

       |

       v

Reuse computed data

CHECKPOINT

       |

       v

Save an intermediate result

       |

       v

Truncate previous lineage

       |

       v

Continue from the checkpoint
```

**The production rule:** Use caching when repeated computation is expensive. Use checkpointing when the dependency chain itself becomes a problem.

- **Caching in Apache Spark:** Reuse computed data to avoid unnecessary recomputation.
- **Persistence in Apache Spark:** Choose storage levels for retaining intermediate data.
- **Spark Architecture:** Understand how Spark executes jobs, stages, and tasks.
- **Lazy Evaluation in Apache Spark:** Understand when transformations actually execute.

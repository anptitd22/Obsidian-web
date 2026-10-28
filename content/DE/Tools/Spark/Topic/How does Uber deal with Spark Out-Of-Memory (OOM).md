![Post cover image](https://freedium-mirror.cfd/img/medium/700/0*i9MXGq1n1zh-mXaY.png)

> *To sustain my writing, I put this article behind a paywall; if you have a Medium membership, I really appreciate it. If you prefer to read this for FREE, please visit [here](https://open.substack.com/pub/vutr/p/free-how-does-uber-deal-with-spark?r=2rj6sg&utm_campaign=post&utm_medium=web). Also, I post only 1/4 of my deep-dive data engineering article on Medium; for the full repo: [https://vutr.substack.com/](https://vutr.substack.com/)*

### Intro

Uber, famous for their high-volume data processing, encounters the same Spark Out-Of-Memory (OOM) errors that many of us face in production.

I got a chance to read an article they shared about how they deal with it.

It's very interesting, and I believe you will enjoy it as well.

So, here we are.

What you're gonna read next is what Spark OOM is, why it happens, and how Uber deals with it.

I won't spend too much time explaining how Spark works in this article; for that purpose, you can visit my previous article [here](https://vutr.substack.com/p/if-youre-learning-apache-spark-this).

> **Disclaimer**: the technical details on how Uber solve the OOM problem come entirely from their blog airticle: [Dynamic Executor Core Resizing in Spark](https://www.uber.com/us/en/blog/dynamic-executor-core-resizing-in-spark/) (2023)

### How does Spark work?

If you want to run Spark, you must have a cluster of machines that provide the resources for the Spark cluster.

![None](https://freedium-mirror.cfd/img/medium/4000/0*21HVc_z-I4esO11q.png)

A Spark cluster is a set of JVM processes, including a Driver and Executors. Those processes run on the cluster of machines (with communication with the Cluster Manager).

![None](https://freedium-mirror.cfd/img/medium/4000/0*qWb1dIitE5hwAxSS.png)

Every Spark cluster is associated with a Spark application.

Below the application is the Spark job. A job represents a series of transformations applied to data: the entire workflow from start to finish. The series of transformations (e.g, filter, map…) can be triggered only by an action (e.g., show, count,…). We can say that a job is associated with an action. An application can have multiple jobs.

![None](https://freedium-mirror.cfd/img/medium/4000/0*y4C_w4ulL4GH219x.png)

A job is split into different stages when a transformation requires shuffling data across partitions. (e.g., groupBy, join). A stage is a job segment executed without data shuffling.

A stage has a set of tasks. A task is the smallest unit of execution within Spark. Each stage is divided into multiple tasks, each handling a partition, a portion of data from an external source or from the upstream stage.

![None](https://freedium-mirror.cfd/img/medium/4000/0*y0PKus8t6GF88ld7.png)

At a given stage, tasks can run in parallel; the parallelism depends on the executor's CPUs. You can understand that tasks are handled in parallel in an executor using the multithreading paradigm. By default, a task is handled by an executor core (controlled by the "spark.task.cpus" setting); if the executor has 4 cores, 4 tasks can run in parallel within the executor.

### Why do OOM errors happen?

However, the executor memory cannot be clearly separated like the cores; you might be aware of this: in multi-thread processing, all threads share the memory. With task-processing in an executor, all tasks share the executor's processing memory.

![None](https://freedium-mirror.cfd/img/medium/4000/0*VVca5NiPSNl5PLVg.png)

Assuming an executor with 4 cores and 8 GB of RAM available for processing and storing data, the executor can handle 4 tasks in parallel, and each task is expected to use up to 2 GB of RAM.

![None](https://freedium-mirror.cfd/img/medium/4000/0*47lkCB8uuYO-z_7H.png)

If a task needs 5GB of RAM to handle, it causes an OOM.

Thus, there are two main reasons why OOM errors happen:

- A task requires more memory to handle a partition.
- The memory portion of a single task shrinks.

For the first one, many factors could cause this: a task is assigned a skewed partition, the input data volume increases, an operation, such as aggregation, requires memory to hold aggregated values, and it is somehow exploited.

For the latter, the most common root cause is increasing the parallelism of an executor but keeping the memory the same (increase the spark.executor.cores but not change the spark.executor.memory)

An annoying thing is that the OOM might not be deterministic. Back to the example above, a task that handles 5GB of data causes OOM when running alongside three other 3 tasks that only need at most 2GB of memory.

There is a case when the OOM won't happen: when the heavy task runs alone or with at most a single normal task at the same time in the executor:

![None](https://freedium-mirror.cfd/img/medium/4000/0*9Qrtaw-8-rDnoyjY.png)

- If the heavy task runs alone, all 8GB of memory is available for it, and it only needs 5GB to do well.

![None](https://freedium-mirror.cfd/img/medium/4000/0*7JnahrjvxUOyRFm0.png)
- If the heavy task runs with only one normal task, both only require 7GB of data, still below the 8GB cap.

If the Spark cluster has two executors, each has 4 cores, this means 8 tasks can run at the same time.

If the stage's total tasks are not divided by 8, for example, 19 or 20, there is a point in time where the total 3 or 4 tasks are run at the same time; each executor could handle 1 or 2 tasks at a time.

If the heavy task somehow ends up in these runs, OOM won't occur.

### Spark Scheduler behavior

You might wonder, "Why doesn't the Spark scheduler check the partition's size to allocate it more cleverly, e.g., run the heavy task alone? "

This is because the scheduler wasn't designed like that. It is built to maximize the Spark cluster resource utilization.

When a ready-to-run task set is available, the scheduler examines available task slots (executors' cores) and fills them as quickly as possible, without using the partition's size information.

![None](https://freedium-mirror.cfd/img/medium/4000/0*zIly6ZzfPUoCwXnM.png)

Back to the above example with an executor with 4 cores and 8GB of memory. The scheduler sees 4 open slots and assigns 4 tasks. 3 tasks are normal, one of them has a skewed partition and needs 5GB on its own.

The scheduler's assumption about resource usage is fine when all tasks require the same CPU and memory. However, when a small percentage of tasks require 3x or 5x as much memory as the rest, the assumption no longer holds.

Neither the scheduler nor the executor knows about this beforehand. So all 4 tasks start running together, sharing the 8GB memory pool. The heavy task keeps asking for more. At some point, the pool runs out. The task failed first, then the executor.

![None](https://freedium-mirror.cfd/img/medium/4000/0*jAtq7P2bq_80YZsu.png)

When a task fails, if the number of failures is still within the threshold (spark.task.maxFailures; default: 4), the scheduler resets the task's state and places it in the pending queue. Later, the failed task is executed again, but the problem is not resolved if it's still run with other tasks. If you are lucky, the task is re-run when the number of tasks is exhausted.

![None](https://freedium-mirror.cfd/img/medium/4000/0*XHcAtzmq6GDe11ci.png)

As I mentioned above, if you have two executors with 4 cores each, 8 tasks can run in parallel. If the total number of tasks is not divisible by 8, such as 19 or 20, some of the last tasks will run with only 3 or 4 tasks at a time instead of 8. This means that if you retry the task, there's a high chance it will succeed.

This is why the same job can pass on Monday and fail on Thursday. A different scheduling order will give a different outcome.

That's the root of why Spark OOM errors sometimes feel unpredictable.

### How does Uber deal with OOM?

After understanding the nature of the OOM error and the scheduler behavior, let's find out how Uber deals with it.

Uber tries to bring predictability to the table here.

They adjust the scheduler so it now tracks the tasks run on an executor. The target is to identify the memory-intensive tasks, the ones that cause OOM.

![None](https://freedium-mirror.cfd/img/medium/4000/0*cvCOnTb5NQOpUmCh.png)

To make things simple, Uber marks all tasks that run on the failed executor as memory-intensive (although one of those tasks is the root cause)

With the target tasks in place, things are straightforward from here.

As discussed in the previous section, OOM occurs because the assumption that all tasks in an executor require the same amount of CPU and RAM no longer holds.

An executor with 4 cores and 8GB of RAM.

Three require 2GB of RAM.

One requires 5GB of RAM.

The whole executor will go down.

Dynamic Executor Core Resizing marks all these 4 tasks memory-intensive, then simply retry, with a twist here: each task will have the resources of the full executor

![None](https://freedium-mirror.cfd/img/medium/4000/0*Po1lyv3Cem1vjNV_.png)

The three tasks that require 2GB of RAM each (of course) run fine on the 4-core, 8GB-RAM executor.

For the big one that requires 5GB of RAM, because it doesn't need to share the executor's resources with others now, it can have 5GB of RAM as it wishes.

Uber trades off a bit of resource utilization here in exchange for reliability: the feature doesn't require human intervention to adjust the executor's resources after an OOM occurs; the scheduler will automatically retry in a "smarter" way.

This helps Uber avoid relying on luck when retrying to get over the OOM; the task that caused the OOM is rerun on the executor alone, so Uber forces it now.

In return, the executor's resource utilization is lower in Spark's original design: back to the example above, when a task requiring 2GB of RAM runs alone on the executor, the remaining 3 cores and 6GB of RAM sit idle.

![None](https://freedium-mirror.cfd/img/medium/4000/0*i-tPbXpUiV91wbjX.png)

Not leveraging all of the executor's resources at a time could mean the end-to-end duration of a Spark job increases.

One thing to note here is that although memory-intensive tasks now get the full executor, they can still fail; in the example above, if the "big" task requires 10 GB instead of 5GB of RAM, the executor's resources are still not enough.

At least Uber has identified the "big" task here and can apply optimizations, such as salting.

### Outro

In this article, I share my understanding of what a Spark OOM error is, why it happens, and, finally, how Uber deals with it using a simple engineering approach while providing much greater reliability.

Thank you for reading this far.

See you in my next article.

### Reference

\[1\] Kalyan Sivakumar, [Dynamic Executor Core Resizing in Spark](https://www.uber.com/us/en/blog/dynamic-executor-core-resizing-in-spark/) (2023)

| **Chiến lược Join**                   | **Điều kiện kích hoạt**                                 | **Shuffle mạng?** | **Độ phức tạp / Chi phí chính**                   | **Rủi ro chính**                                           |
| ------------------------------------- | ------------------------------------------------------- | ----------------- | ------------------------------------------------- | ---------------------------------------------------------- |
| **Broadcast Hash Join (BHJ)**         | Equi-join (`=`), 1 bảng $\le$ 10MB (hoặc có hint)       | **Không**         | Rất nhanh ($O(M + N)$), tốn bộ nhớ lưu Hash Table | OOM Driver (khi collect) hoặc OOM Executor (khi broadcast) |
| **Shuffle Hash Join (SHJ)**           | Equi-join (`=`), bảng trung bình/lớn, partition vừa RAM | **Có**            | Shuffle I/O mạng + CPU build Hash Table           | OOM Executor nếu kích thước partition sau shuffle vượt RAM |
| **Shuffle Sort Merge Join (SMJ)**     | Equi-join (`=`), cả 2 bảng đều rất lớn (Default)        | **Có**            | Shuffle I/O mạng + CPU Sort 2 bên                 | Chậm do Shuffle & Sort; dễ gặp nghẽn do Data Skew          |
| **Broadcast Nested Loop Join (BNLJ)** | Non-equi join (`<`, `>`, `BETWEEN`, `crossJoin`)        | **Không**         | Cực kỳ tốn CPU ($O(M \times N)$)                  | Job bị treo/chạy vô tận; OOM khi broadcast                 |

![Post cover image](https://freedium-mirror.cfd/img/medium/700/1*PVrmLteqaYP2tLHAAxxyMQ.png)
### What is a Broadcast Hash Join?

In Apache Spark, a **Broadcast Hash Join (BHJ)** is a highly optimized join strategy used when one of the tables being joined is significantly smaller than the other. Instead of shuffling data from both tables across the network to match keys (which is extremely expensive), Spark "broadcasts" the entire smaller table to every worker node (executor) in the cluster.

By having a full copy of the small table locally on every node, Spark can join it against the partitions of the larger table directly, completely eliminating the need for a network shuffle.

### How it Works (The Execution Flow)

The execution of a Broadcast Hash Join happens in three distinct phases:

![None](https://freedium-mirror.cfd/img/medium/4000/1*0l5Ofjx1RxKSA_-t_3px2Q.png)

**1\. The Collection Phase (Driver)**

- Spark's Catalyst Optimizer identifies that one table is small enough to be broadcasted (based on statistics and configuration).
- The Spark Driver collects all the data from the smaller table into its own memory.

**2\. The Build & Broadcast Phase**

- The Driver builds a hash table in memory using the join key from the small table.
- It then broadcasts this serialized hash table to all the Executors in the cluster. Spark uses a highly efficient, BitTorrent-like peer-to-peer protocol to distribute this data quickly, preventing the Driver from becoming a network bottleneck.

**3\. The Probe Phase (Executors)**

- Each Executor holds a read-only copy of the small table's hash table in its memory.
- The Executors then read their assigned partitions of the **large table** row by row.
- For each row of the large table, the Executor computes the hash of the join key and "probes" the local hash table. If there is a match, the joined row is immediately outputted.

### Key Configurations and Triggers

Catalyst will automatically choose a Broadcast Hash Join if the size of the smaller table is below a specific threshold.

- `spark.sql.autoBroadcastJoinThreshold`: This is the configuration property that dictates the maximum size of a table that will be auto-broadcasted.
- **Default Value**: It is traditionally set to **10 MB**.
- **Manual Override**: You can force Catalyst to attempt a broadcast join using SQL hints, like `/*+ BROADCAST(small_table) */`, even if the table size is slightly above the threshold or statistics are missing.

### Advantages vs. Risks

**The Good:**

- **Zero Shuffle:** This is the fastest join strategy in Spark because it completely bypasses the shuffle phase, saving massive amounts of network I/O, disk I/O, and serialization overhead.
- **Handles Data Skew:** Because the large table isn't shuffled by key, data skew in the large table doesn't cause out-of-memory (OOM) errors or straggler tasks.

**The Risks (Why we don't broadcast everything):**

- **Driver OOM:** If the "small" table is actually quite large (e.g., you forced it with a hint), collecting it to the Driver can cause a `java.lang.OutOfMemoryError` on the Driver node, killing the entire application.
- **Executor OOM:** The broadcasted hash table must fit entirely into the Execution Memory of every Executor. If it doesn't, the Executors will crash.

### What is a Shuffle Sort Merge Join?

![None](https://freedium-mirror.cfd/img/medium/4000/1*SXLitXSMJF-YDaLMp0KVbQ.png)

In Apache Spark, a **Shuffle Sort Merge Join (SMJ)** is the heavy-duty, default join strategy used when you are joining two massive tables together. Unlike a Broadcast Hash Join, which avoids network movement by copying a small table everywhere, SMJ assumes both tables are too large to fit entirely into an executor's memory.

To perform this join, Spark must reorganize the data across the cluster, sort it locally on the nodes, and then merge the matching records.

### How it Works (The Execution Flow)

The execution of a Shuffle Sort Merge Join happens in three distinct, resource-intensive phases:

**1\. The Shuffle Phase (Network Exchange)**

- Spark reads both tables and computes a hash of the join keys.
- It then physically moves (shuffles) the data across the cluster's network so that rows with the *same* hash (and therefore the same join key) from *both* tables end up perfectly co-located on the exact same partition of the same Executor.
- This is the most expensive part of the operation, requiring heavy network I/O and disk I/O as data is written and read from shuffle files.

**2\. The Sort Phase**

- Once the shuffled data arrives at its destination partition, the Executor sorts the data from both tables independently based on the join key.
- This sorting happens locally at the partition level.

**3\. The Merge Phase**

- With both sides of the data now residing on the same node and perfectly sorted, the Executor begins the merge.
- It uses an efficient "two-pointer" algorithm, streaming the sorted data row by row. Since the keys are in order, the pointer simply marches down both lists, stopping to output a joined row whenever the keys from both tables match.

### Advantages vs. Costs

**The Good (Why Spark uses it for large tables):**

- **Massive Scalability:** It can join datasets of virtually any size.
- **Low Memory Footprint:** Because the data is sorted, the Merge phase only needs to keep a tiny amount of data in memory at a time. It doesn't need to build massive, memory-hogging hash tables, which effectively prevents Executor Out-Of-Memory (OOM) errors during the actual join computation.

**The Costs:**

- **High Latency:** Moving massive amounts of data across the network during the shuffle is inherently slow.
- **CPU Intensive:** Sorting millions or billions of rows requires significant CPU cycles.

### Key Configurations

- `spark.sql.join.preferSortMergeJoin`: By default, this is set to `true`. Catalyst will favor this robust strategy for large tables unless one table is small enough to trigger a Broadcast Hash Join.
- **Note on Modern Spark (3.x+):** If you have Adaptive Query Execution (AQE) enabled, Spark might dynamically downgrade a planned Sort Merge Join into a Broadcast Hash Join or a Shuffle Hash Join at runtime if it realizes the data sizes are smaller than initially expected after the shuffle phase.

### 1\. Shuffle Hash Join (SHJ)

![None](https://freedium-mirror.cfd/img/medium/4000/1*y70qKzryDpZzgHW9CZCTug.png)

While SMJ is the default for large tables, **Shuffle Hash Join** is a powerful alternative for medium-to-large tables.

**When it's used:** It is ideal when one table is too large to broadcast, but its individual *partitions* are small enough to fit into executor memory after a shuffle.

**How it works:**

- Spark shuffles both tables using the join key (just like in SMJ).
- Instead of sorting the data, Spark takes the smaller partition on the executor and builds an in-memory hash table out of it.
- It then streams the larger partition, probing the hash table to find matches.

> **Why interviewers ask about it:** Sorting (used in SMJ) is highly CPU-intensive. Building a hash table is faster. In modern Spark (with Adaptive Query Execution), Spark will dynamically switch from an SMJ to an SHJ at runtime if it detects that the post-shuffle partitions are small enough to fit in memory, saving precious CPU cycles.

![None](https://freedium-mirror.cfd/img/medium/4000/1*liTIAwPDHrlyzxkI29hcLw.png)

### What is a Broadcast Nested Loop Join (BNLJ)?

In Apache Spark, a **Broadcast Nested Loop Join (BNLJ)** is the brute-force, fallback join strategy. While a Broadcast Hash Join (BHJ) builds a fast, in-memory hash table for instant lookups, BNLJ simply broadcasts the raw data of the smaller table to all executors and uses a massive "nested loop" to compare every single row of the large table against every single row of the small table.

It is notoriously slow and CPU-intensive ***O(M X N)*** time complexity, but it is necessary because it is the only join strategy capable of handling complex, non-standard join conditions where hashing or sorting is mathematically impossible.

### How it Works (The Execution Flow)

The execution of a BNLJ happens in two primary phases:

![None](https://freedium-mirror.cfd/img/medium/4000/1*NwSPaxjFXQBhesfyAEJ_SQ.png)

**1\. The Collection & Broadcast Phase (Driver)**

- Similar to BHJ, Spark's Catalyst Optimizer identifies one table to be broadcasted (usually the smaller one).
- The Spark Driver collects the data from this table into its memory. However, unlike a BHJ, the Driver **does not** build a hash table. It just holds the raw array of rows.
- It broadcasts this raw array to all Executor nodes in the cluster.

**2\. The Nested Loop Phase (Executors)**

- Each Executor now holds a full, read-only copy of the small table in its memory.
- The Executor begins iterating through its assigned partition of the **large table** row by row (the Outer Loop).
- For *every single row* of the large table, the Executor must iterate through *every single row* of the broadcasted small table (the Inner Loop) and evaluate the custom join condition to see if it returns `true`.

### When Spark Uses It (The Triggers)

Spark avoids BNLJ whenever possible. Catalyst will only force a BNLJ when you are performing a **Non-Equi Join**.

If your join condition does not use a strict equality operator (==), Spark cannot compute a hash or sort the keys. Examples include:

- **Range Joins:** `tableA.ip_address BETWEEN tableB.start_ip AND tableB.end_ip`
- **Inequality Joins:** `tableA.date < tableB.expiration_date`
- **Cartesian Products:** `df1.crossJoin(df2)`

### Advantages vs. Costs

**The Good:**

- **Ultimate Flexibility:** It is the "join of last resort." If you have a highly complex, multi-conditional, or purely comparative join logic, BNLJ guarantees that Spark can actually compute the answer.
- **No Network Shuffle:** Like BHJ, it completely avoids shuffling the massive table across the cluster network.

**The Costs (The Risks):**

- **Massive CPU Overhead:** If your large table has 1 billion rows, and your broadcasted small table has 10,000 rows, a BNLJ requires **10 trillion** individual row comparisons on your CPUs. This can cause jobs to hang for hours.
- **OOM Vulnerability:** Just like BHJ, if the table chosen for broadcasting is too large, it will trigger an `OutOfMemoryError` on the Driver during the collection phase, or on the Executors during the broadcast phase.

### 2\. AQE Skew Join Optimization

"Data Skew" is practically a guaranteed topic in any Spark interview. Skew happens when your join key is unevenly distributed (e.g., joining on `country_code` where 90% of your users are from the US). In a standard Shuffle Sort Merge Join, one executor gets stuck processing the massive "US" partition, causing a "straggler task" or an Out-Of-Memory error.

**The Modern Solution:** Adaptive Query Execution (AQE) handles this automatically at runtime.

**How it works:**

1. Spark pauses after the shuffle phase and looks at the partition sizes.
2. If it detects a heavily skewed partition, AQE splits that massive partition into several smaller sub-partitions.
3. To ensure no data is lost during the join, Spark duplicates the corresponding join keys from the *other* table so they can be joined against all the new sub-partitions.

> **Why interviewers ask about it:** In the past (Spark 2.x), handling skew required manual, painful workarounds like "salting" (adding random numbers to keys). Knowing that AQE handles this natively shows you are up-to-date with current Spark architecture.

![None](https://freedium-mirror.cfd/img/medium/4000/1*fzfZFH_1SsuQ1YzOc8Yauw.png)

### The Decision Hierarchy (How Catalyst Thinks)

When Spark encounters a standard equi-join (e.g., `ON tableA.id = tableB.id`), the Catalyst Optimizer evaluates the strategies in this exact order:

1. **Can I Broadcast it?** Catalyst checks if the estimated size of either table is under 10 MB. If yes, it plans a **Broadcast Hash Join**.
2. **Can I Hash it locally?** If neither table is small enough to broadcast, but one table's partitions will fit in memory, it *might* plan a **Shuffle Hash Join** (highly dependent on AQE and configurations).
3. **I must Sort and Merge.** If both tables are massive, Catalyst defaults to its most robust, scalable strategy: the **Shuffle Sort-Merge Join**.
4. **Wait, let me check again (AQE).** During execution, AQE pauses after the data is shuffled across the network. If the actual data size is much smaller than Catalyst originally guessed, AQE interrupts the plan and dynamically switches to a BHJ or SHJ to speed things up.

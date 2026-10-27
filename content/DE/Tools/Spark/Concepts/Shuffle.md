
- **Mục đích:** Tái phân phối dữ liệu qua mạng (cluster network) sao cho **các record có cùng key nằm trên cùng một partition**.
    
- **Đặc tính:** Xuất hiện ở các **Wide Transformation** (`groupBy`, `join`, `distinct`, `repartition`). Tạo ra ranh giới giữa 2 Stage.

#### The Complete Shuffle Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. STAGE 1 COMPLETES (Shuffle Write)                        │
│    - Tasks process data and write shuffle files to disk     │
│    - Executors report file locations to driver              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. DRIVER PLANS STAGE 2                                     │
│    - Driver knows shuffle data locations                    │
│    - Driver creates tasks for Stage 2                       │
│    - Driver assigns partitions to tasks                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. STAGE 2 STARTS (Shuffle Read)                            │
│    - New tasks ask driver for shuffle data locations        │
│    - Tasks fetch data from executors                        │
│    - Tasks process the data                                 │
└─────────────────────────────────────────────────────────────┘
```

Let me break down each step in detail.

#### When Does Spark Know Partition and Task Counts?

**Timeline of Planning and Execution**

```
TIME 0: You submit code
  ↓
TIME 1: Driver creates LOGICAL PLAN
  - Knows operations: filter, groupBy, etc.
  - Doesn't know partitions yet
  ↓
TIME 2: Driver creates PHYSICAL PLAN
  ✓ NOW Spark knows partition counts for ALL stages
  ✓ NOW Spark knows task counts for ALL stages
  - Decides: Stage 1 = 100 partitions = 100 tasks
  - Decides: Stage 2 = 200 partitions = 200 tasks
  ↓
TIME 3: Driver launches Stage 1 tasks
  - Creates 100 task objects
  - Sends tasks to executors
  - Tasks start running
  ↓
TIME 4: Stage 1 tasks complete
  - Tasks finish and write shuffle files
  ↓
TIME 5: Driver launches Stage 2 tasks
  - Creates 200 task objects
  - Sends tasks to executors
  - Tasks start running
```
#### How Does Spark Decide Partition Counts?

**For Stage 1 (before shuffle):**

```
# Reading from files
df = spark.read.parquet("data.parquet")
# Partitions = number of files (e.g., 100 files = 100 partitions)
```

**For Stage 2 (after shuffle):**

```
# Default configuration
spark.sql.shuffle.partitions = 200  # Default value

# This means ANY shuffle operation creates 200 partitions in the next stage
df.groupBy("customer_id").sum("amount")  # Stage 2 will have 200 partitions

# You can change it
spark.conf.set("spark.sql.shuffle.partitions", 500)  # Now Stage 2 has 500 partitions
```

**Key Point:** The driver decides how many tasks will be in every stage **before any tasks run.** It does it during the physical planning phase.

#### When Are Tasks Created?

**Stage 1 tasks:** Created immediately when the job starts **(TIME 3)**

**Stage 2 tasks:** Created **after Stage 1 completes (TIME 5)**

- Driver doesn't create **Stage 2 tasks** until **Stage 1** has data output in the disk
- **But driver already knows there how many tasks will be in stage 2** (**decided at TIME 2**)

### Phase 1: Shuffle Write (End of Stage 1)

#### What Happens

**Stage 1** is running. You have multiple tasks processing partitions. When these tasks complete their work (all narrow transformations), they need to prepare data for the next stage.

#### Step 1: Determine Output Partitions

Before **Stage 1 tasks** even start, the driver has already decided: *"Stage 2 will have 200 partitions".*

This number comes from `spark.sql.shuffle.partitions = 200`.

**Why does Stage 1 need to know this?**

> Because each **Stage 1 task** needs to organize its output into 200 separate buckets (**one for each Stage 2 partition**).

#### Step 2: Decide Where Each Record Goes

Now **Stage 1 tasks** are running. Let's say **Task 5** in **Stage 1** is processing its partition and encounters this record:

```
("customer_A", 10)
```

The task asks itself: "I have this `customer_A` record. **Stage 2** will have **200 partitions**. Which one of those **200 partitions** should this record go to?"

**How it decides (Hash Partitioner):**

```
# The task runs this calculation:
partition_number = hash("customer_A") % 200

# Let's say the hash function returns 9342
# 9342 % 200 = 42

# Result: This record goes to partition 42 of Stage 2
```

**What is hash()?**

A hash function converts any value (like " `customer_A` ") into a number. Same input always gives same output.

**What is % (modulo)?**

It gives the remainder after division. `9342 % 200 = 42` because *9342 ÷ 200 = 46 remainder 42.*

**Why this matters?**

Every time ANY task in **Stage 1** sees " `customer_A` ", it calculates the same hash and gets the same result (42). This ensures ALL `customer_A` records end up in **partition 42** of **Stage 2**.

**Example with multiple records:**

```
Task 5 in Stage 1 processes these records:

Record: ("customer_A", 10)
  → hash("customer_A") % 200 = 42  → Goes to partition 42

Record: ("customer_B", 20)
  → hash("customer_B") % 200 = 105 → Goes to partition 105

Record: ("customer_A", 15)
  → hash("customer_A") % 200 = 42  → Goes to partition 42 (same as before!)

Record: ("customer_C", 30)
  → hash("customer_C") % 200 = 42  → Goes to partition 42 (happens to be same)
```

So **Task 5** now knows:

- **Partition 42** should get: `customer_A` ($10), `customer_A` ($15), `customer_C` ($30)
- **Partition 105** should get: `customer_B` ($20)

#### Step 3: Write Shuffle Files to Disk

**What is a shuffle file?**

> *A **shuffle file** is a regular file on the **executor's local disk** that stores the output data from Stage 1, organized by which Stage 2 partition it belongs to.*

Now **Task 5** writes files. It creates **one file per Stage 2 partition that has data**.

**Example:**

```
Stage 1 has 3 tasks (Task 0, Task 1, Task 2)
Stage 2 will have 2 partitions

Task 0 in Stage 1 writes:
  /tmp/spark/shuffle_0_0.data  → All records that go to Stage 2 partition 0
  /tmp/spark/shuffle_0_1.data  → All records that go to Stage 2 partition 1

Task 1 in Stage 1 writes:
  /tmp/spark/shuffle_1_0.data  → All records that go to Stage 2 partition 0
  /tmp/spark/shuffle_1_1.data  → All records that go to Stage 2 partition 1

Task 2 in Stage 1 writes:
  /tmp/spark/shuffle_2_0.data  → All records that go to Stage 2 partition 0
  /tmp/spark/shuffle_2_1.data  → All records that go to Stage 2 partition 1

Total: 6 files (3 tasks × 2 partitions)
```

**Why disk?** Because:

1. **Stage 1 tasks might finish before Stage 2 tasks start** → Data needs to persist somewhere
2. **Stage 1 and Stage 2 tasks might run on different executors** → Need a reliable storage location
3. **Memory is limited** → Disk can hold much more data
4. **Fault tolerance** → If Stage 2 fails, we can retry without recomputing Stage 1

#### Step 4: Register Metadata with Driver

**What is metadata?**

**Metadata is information ABOUT the data**, it is not the data itself. It includes file locations, sizes, executor addresses — small descriptive information.

After writing files, each executor reports to the driver:

```
Executor 3 → Driver: "Task 0 completed. Shuffle files at:
  - Executor address: executor-3
  - File path: /tmp/spark/shuffle_0_0.data (size: 1.2MB)
  - File path: /tmp/spark/shuffle_0_1.data (size: 800KB)"
```

The driver stores this in **MapOutputTracker**, which is a lookup table.

**What is MapOutputTracker?**

A registry (like a phone book) maintained by the driver that tracks where every shuffle file is located across all executors.

```
MapOutputTracker contents:
Shuffle ID 0:
  Task 0 → Executor 3, files: [shuffle_0_0.data, shuffle_0_1.data]
  Task 1 → Executor 5, files: [shuffle_1_0.data, shuffle_1_1.data]
  Task 2 → Executor 7, files: [shuffle_2_0.data, shuffle_2_1.data]
```

**Important:** This metadata is tiny (just file paths and sizes), but the actual data is large and stays on executor disks.

### Phase 2: Driver Plans Stage 2

#### When Are Stage 2 Tasks Created?

**Answer: After Stage 1 completes.**

**Here's the timeline:**

1. **Before job starts:** Driver already knows there will be a **Stage 2 with 200 partitions**
2. **Stage 1 runs:** Tasks write shuffle files
3. **Stage 1 tasks complete:** Executors report file locations to driver
4. **Driver creates Stage 2 tasks:** Now that data is ready, driver creates 200 tasks (one per partition)
5. **Stage 2 starts:** Tasks are sent to executors

#### How Does Driver Know How Many Tasks to Create?

**Simple answer:** One task per partition.

```
If Stage 2 has 200 partitions → Create 200 tasks
```

#### How Does Driver Know Which Data Each Task Needs?

The driver uses the **MapOutputTracker** metadata:

> **Task 42 in Stage 2 is responsible for partition 42**

**What does "responsible for partition 42" mean?**

**Task 42's job** is to fetch ALL the data that belongs to **partition 42 (from all Stage 1 tasks)** and process it.

```
Driver logic:
"Task 42 needs partition 42 data. Let me check which Stage 1 tasks wrote partition 42 files..."

MapOutputTracker lookup:
  - Task 0 wrote shuffle_0_42.data on Executor 3
  - Task 1 wrote shuffle_1_42.data on Executor 5
  - Task 2 wrote shuffle_2_42.data on Executor 7

Driver tells Task 42:
"Your input data is:
  - shuffle_0_42.data from Executor 3
  - shuffle_1_42.data from Executor 5
  - shuffle_2_42.data from Executor 7"
```

**This information is embedded in the task itself** when the driver creates it. The task carries this metadata as part of its definition.

### Phase 3: Shuffle Read (Start of Stage 2)

#### Step 1: Task Receives Assignment

When **Stage 2 Task 42** starts on an executor, it already knows:

```
I need to process partition 42.

My input files are:
  - shuffle_0_42.data on Executor 3
  - shuffle_1_42.data on Executor 5
  - shuffle_2_42.data on Executor 7
```

#### Step 2: Fetch Shuffle Blocks

**What is a shuffle block?**

A **shuffle block** is a chunk of data from a **shuffle file**. It's the actual data that needs to be transferred and processed.

The task fetches data from executors:

**Scenario A: Local Read (same executor)**

```
Task 42 runs on Executor 3
shuffle_0_42.data is also on Executor 3
→ Read directly from local disk (fast!)
```

**Scenario B: Remote Read (different executor)**

```
Task 42 runs on Executor 3
shuffle_1_42.data is on Executor 5
→ Request file over network:
  1. Task 42 sends HTTP/RPC request to Executor 5's BlockManager
  2. Executor 5's BlockManager reads file from its local disk
  3. Executor 5's serialize the data
  4. Executor 5 streams data over network to Task 42
  5. Task 42 receives data in memory buffers
```

**What is BlockManager?**

A component on each executor that **manages data storage and retrieval**. It serves shuffle files to other executors when requested.

> **Parallelism:** Task 42 fetches from multiple executors simultaneously.

#### Step 3: Merge and Process

As data arrives, **Task 42**:

1. **Deserializes** the data (converts bytes back to objects)
2. **Merges** streams from different sources
3. **Sorts** by key if needed (for operations like `sortByKey`)
4. **Applies the transformation** (e.g., `sum`, `count`, `join`)

**What is deserialize?**

When data is written to disk or sent over network, it's converted to bytes (serialized). Reading it back requires converting bytes to usable objects (deserialized).

**Example for** `groupBy().sum()`:

```
Task 42 receives:
  From shuffle_0_42.data: (customer_A, 10), (customer_A, 15)
  From shuffle_1_42.data: (customer_A, 30), (customer_C, 40)
  From shuffle_2_42.data: (customer_A, 5)

Task 42 aggregates:
  customer_A: 10 + 15 + 30 + 5 = 60
  customer_C: 40

Output: [(customer_A, 60), (customer_C, 40)]
```

### Complete Example: End-to-End

Let's trace `df.groupBy("customer_id").sum("amount")` with real numbers.

#### Initial Setup

```
# Configuration
# Configuration
spark.sql.shuffle.partitions = 2  # Stage 2 will have 2 partitions

# Input data (3 partitions in Stage 1)
Partition 0: [(A, 10), (B, 20)]
Partition 1: [(A, 30), (C, 40)]
Partition 2: [(A, 5), (B, 15)]
```

#### Stage 1: Shuffle Write

**Task 0 in Stage 1 (processes partition 0):**

```
For (A, 10):
  hash("A") % 2 = 0  → This record goes to Stage 2 partition 0
  → Write to shuffle_0_0.data

For (B, 20):
  hash("B") % 2 = 1  → This record goes to Stage 2 partition 1
  → Write to shuffle_0_1.data

Files created on Executor 3:
  shuffle_0_0.data: (A, 10)
  shuffle_0_1.data: (B, 20)

Executor 3 → Driver: "Task 0 done, files on Executor 3"
```

**Task 1 in Stage 1 (processes partition 1):**

```
For (A, 30):
  hash("A") % 2 = 0  → This record goes to Stage 2 partition 0
  → Write to shuffle_1_0.data

For (C, 40):
  hash("C") % 2 = 0  → This record goes to Stage 2 partition 0
  → Write to shuffle_1_0.data

Files created on Executor 5:
  shuffle_1_0.data: (A, 30), (C, 40)
  shuffle_1_1.data: (empty file, no data for partition 1)

Executor 5 → Driver: "Task 1 done, files on Executor 5"
```

**Task 2 in Stage 1 (processes partition 2):**

```
For (A, 5):
  hash("A") % 2 = 0  → This record goes to Stage 2 partition 0
  → Write to shuffle_2_0.data

For (B, 15):
  hash("B") % 2 = 1  → This record goes to Stage 2 partition 1
  → Write to shuffle_2_1.data

Files created on Executor 7:
  shuffle_2_0.data: (A, 5)
  shuffle_2_1.data: (B, 15)

Executor 7 → Driver: "Task 2 done, files on Executor 7"
```

**Driver State After Stage 1**

```
MapOutputTracker:
  Stage 2 Partition 0 data is in:
    - shuffle_0_0.data on Executor 3
    - shuffle_1_0.data on Executor 5
    - shuffle_2_0.data on Executor 7

  Stage 2 Partition 1 data is in:
    - shuffle_0_1.data on Executor 3
    - shuffle_1_1.data on Executor 5
    - shuffle_2_1.data on Executor 7
```

**Driver Creates Stage 2**

```
Create 2 tasks (one per partition):
  Task 0 in Stage 2: Process partition 0
  Task 1 in Stage 2: Process partition 1

Task 0 assignment:
  Fetch and process:
    - shuffle_0_0.data from Executor 3
    - shuffle_1_0.data from Executor 5
    - shuffle_2_0.data from Executor 7

Task 1 assignment:
  Fetch and process:
    - shuffle_0_1.data from Executor 3
    - shuffle_1_1.data from Executor 5
    - shuffle_2_1.data from Executor 7
```

#### Stage 2: Shuffle Read

**Task 0 in Stage 2 (assigned to Executor 3):**

```
Fetches:
  - shuffle_0_0.data from local disk: (A, 10)
  - shuffle_1_0.data from Executor 5 over network: (A, 30), (C, 40)
  - shuffle_2_0.data from Executor 7 over network: (A, 5)

Aggregates:
  A: 10 + 30 + 5 = 45
  C: 40

Output partition 0: [(A, 45), (C, 40)]
```

**Task 1 in Stage 2 (assigned to Executor 5):**

```
Fetches:
  - shuffle_0_1.data from Executor 3 over network: (B, 20)
  - shuffle_1_1.data from local disk: (empty)
  - shuffle_2_1.data from Executor 7 over network: (B, 15)

Aggregates:
  B: 20 + 15 = 35

Output partition 1: [(B, 35)]
```

#### Final Result

```
Partition 0: [(A, 45), (C, 40)]
Partition 1: [(B, 35)]
```

### Mental Model Checkpoint

1. **Driver plans everything upfront** (knows all partition counts before execution)
2. **Stage 1 tasks** write **shuffle files** (one file per Stage 2 partition)
3. **Executors tell driver: "** *My files are here* " (MapOutputTracker)
4. **Driver creates Stage 2 tasks** with embedded instructions
5. **Stage 2 tasks fetch their assigned files and process data**

### Why Are Shuffles Expensive?

Shuffle is expensive because it involves **three costly operations**: disk I/O, network I/O, and managing many files.

#### 1\. Disk I/O (Reading and Writing)

Every record is written to disk, then read from disk:

```
Stage 1 (Shuffle Write):
  - Process 1TB of data
  - Write 1TB to disk  ← COST #1

Stage 2 (Shuffle Read):
  - Read 1TB from disk  ← COST #2

Total: 2TB of disk I/O for 1TB of data
```

**Disk I/O is slow:**

- SSD: ~500 MB/s
- HDD: ~100 MB/s
- Memory: ~10,000 MB/s (100x faster than disk!)

#### 2\. Network I/O (Data Transfer)

**Data must be transferred across the network:**

```
Example:
Stage 1: 100 tasks running on 10 executors
Stage 2: 200 tasks running on 10 executors

Task 42 in Stage 2 needs data from ALL 100 Stage 1 tasks
→ Must fetch from 10 different executors over the network
```

**Network transfer is expensive:**

```
Shuffling 1TB of data:
  - 1Gbps network = 125 MB/s
  - Time: 1TB ÷ 125 MB/s = ~2.3 hours
  - 10Gbps network = ~14 minutes (still slow!)
```

#### 3\. File Management (Many Small Files)

**The file explosion problem:**

```
Formula:
Number of shuffle files = (Stage 1 tasks) × (Stage 2 partitions)

Example:
  1,000 tasks × 200 partitions = 200,000 files

Real production example:
  10,000 tasks × 2,000 partitions = 20,000,000 files!
```

**Why many files are expensive:**

```
Creating/opening/closing 200,000 files takes time:
  - Operating system must manage file handles
  - File metadata operations (inodes, directory entries)
  - Each file operation has overhead (~1ms per file)
  - 200,000 files × 1ms = 200 seconds just for file operations!
```

> That's why using `coalesce` in Spark is a good idea which can optimize the whole job.

#### Additional costs include:

- Serialization/deserialization operations (CPU cost)
- Stage Synchronization Overhead (**Driver must wait for all Stage 1 tasks to complete before starting Stage 2**)
- **Memory spills to disk** (when shuffle data doesn't fit in memory)
- Garbage Collection (GC) Pressure

**What is garbage collection?**

The JVM (Java Virtual Machine) automatically cleans up unused objects in memory. This process temporarily pauses execution.

```
During shuffle:
  - Creating millions of objects (records, buffers, file handles)
  - More objects = more garbage collection
  - GC pauses = tasks stop working

Heavy shuffle can cause:
  - Frequent GC pauses (1-10 seconds each)
  - Tasks appear "stuck" during GC
```

#### Visual Cost Comparison

```
Without Shuffle (Narrow Transformations):
┌──────────┐
│ Read     │  1x disk read
│ Process  │  Pure CPU (fast)
│ Write    │  1x disk write (if needed)
└──────────┘
Total: Minimal overhead

With Shuffle (Wide Transformations):
┌──────────┐
│ Read     │  1x disk read
│ Process  │  CPU work
│ Write    │  1x disk write (shuffle files)     ← Extra cost
│ Report   │  Metadata to driver                ← Extra cost
│ NewTasks │  New Tasks creation and share      ← Extra cost
│ Fetch    │  Network transfer                  ← Extra cost
│ Read     │  1x disk read (shuffle files)      ← Extra cost
│ Merge    │  CPU + potential memory spills     ← Extra cost
│ Process  │  Final computation
└──────────┘
Total: 5-10x more expensive!
```
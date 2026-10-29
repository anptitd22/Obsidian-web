![Post cover image](https://freedium-mirror.cfd/img/medium/700/1*laaerfQXXqWDuvtd4fa3rQ.png)

![None](https://freedium-mirror.cfd/img/medium/4000/1*MebRxtOjKbBLlTmL6n_b4A.png)

**Projection Pushdown** and **Predicate Pushdown** are important **rule-based optimizations** that the **Catalyst Optimizer** applies to optimize Spark queries and reduce the amount of data that needs to be read and processed.

### Projection Pushdown

Let us define a few terms first

- **Projection** is the process of selecting specific columns from a table.
- **Pushdown** means pushing the operation closer to the source.

**Projection Pushdown** therefore means pushing the column selection down to the source. It is just a fancy word for **Column Pruning**. It means reading only the columns required by a query instead of reading all columns from the underlying data source. Reading unnecessary columns waste resources and degrades performance by increasing **disk I/O, network transfer, and memory consumption!** Implementation comprises of two parts —

**\[1\]** The **onus is partially on us**. When writing queries using DataFrames or SQL, we should select only the columns we actually need.

**\[2\]** The other part is handled by **Catalyst Optimizer (CO).** Spark implements **Lazy Evaluation** which means that the transformations are not computed unless an action is performed. This gives Spark a wholistic view of the transformations we are trying to perform, giving it an opportunity to optimize our code.

- Assume we initially select 100 columns in one transformation, but subsequent transformations use only 5 columns. The **net effect** of the query is that only 5 columns are actually required. CO recognizes this and ensures Spark reads only 5 columns and not 100 columns!

> While we should avoid selecting unnecessary columns in the first place, CO can further optimize the query and eliminate columns that are ultimately not required by the query.

### Predicate Pushdown

Once again, let us define a few terms first

- **Predicate** is just a filter operation in a query.
- **Pushdown** means pushing the operation closer to the source.

**Predicate pushdown** is an optimization where Catalyst Optimizer (CO) pushes the filter conditions to the source, allowing it to filter rows before they are read into memory. A predicate is a filter condition in a query, such as age > 21, used in filter() or WHERE. By applying these filters at source, Spark avoids loading irrelevant rows, thereby reducing I/O and improving performance. This technique is effective with data sources that support filtering.

- **Columnar Formats**: Parquet and ORC, which store metadata to enable efficient filtering. Refer *"How Predicate Pushdown works with parquet?"*
- **Databases**: RDBMS like SQL Server, PostgreSQL, MySQL and so forth execute predicates on the server-side.
- **Partitioned Data**: Partitioning organizes your data into folders based on values in the partitioning column(s). It is kind of a high-level form of *Predicate Pushdown* wherein you embed predicates into actual directory structure. When you write query with predicate on partitioning column, Spark can use that predicate to identify and read only relevant partition, while skipping the rest (aka **Partition Pruning**).

> Predicate pushdown is less effective with row-based formats like CSV or JSON, where full rows must often be read before filtering.

#### How Predicate Pushdown works with parquet?

Instead of reading an entire Parquet file into memory and then discarding rows that do not match, Spark leverages Parquet's rich metadata — such as **min/max statistics, dictionary pages, and bloom filters** — to skip irrelevant data blocks altogether. A basic understanding of the Parquet file layout and how column metadata helps with Predicate Pushdown is helpful and is explained below.

**\[1\] Min/Max Statistics**

- Parquet stores **min/max statistics** for each column chunk at the row-group level (stored in file footer) and can also maintain at the page-level (via optional index pages), enabling Spark to skip irrelevant Row Groups or Data Pages.
- Consider sample query and metadata below. Spark reads the min/max statistics and knows for sure the value 9 can only be present in the row-group 0. Therefore, it will skip row group 1 and 2 and read row-group 0.

![None](https://freedium-mirror.cfd/img/medium/4000/1*jgXamA8vHWOhwdOy-p1qXA.png)

**\[2\] Dictionary Filtering**

- The first page in a column chunk is a **Dictionary Page** where parquet maintains a **dictionary of unique values** for that column chunk. Spark can use this dictionary to see whether a predicate value may be present. If value is not present in the dictionary, Spark skips those **Data Pages**.
- It enables Spark to exclude values that lie between min and max but are not listed in the dictionary.

> **Min/Max Statistics and Dictionary Filtering** — both are **ineffective** for **high-cardinality and unsorted columns**. Too many distinct values will increase the size of dictionary and the benefits go for a toss. Min/max statistics will not be able to eliminate significant row-groups if the data is unsorted and widely spread across row-groups. Bloom filters come to the rescue in these scenarios.

**\[3\] Bloom Filters**

- **Bloom filters** store a compact bitset representation of the unique items in a column chunk. If the bloom filter returns **"definitely no"**, Spark is sure that the values does not exist in a row-group and it skips it. If the filter returns **"probably yes"**, Spark reads the row group normally to find the exact record.
- They are useful for columns with high cardinality and scenarios where filter operations are based on values likely to be absent from dataset.

> I know Bloom Filters need more explanation and require a separate deep dive story altogether. Just remember, Bloom Filters are used by google, instagram, facebook and others to check for the existence of a user name when you try to create a new one! So they are great for point queries.

Ví dụ:

```sql
# Cách 1: Dùng SQL thuần qua spark.sql()
spark.sql("""
    EXPLAIN 
    SELECT * 
    FROM metastore.lakehouse_db.customer_orders 
    WHERE status = 'COMPLETED'
""").show(truncate=False)

# Cách 2: Dùng trực tiếp hàm .explain() của DataFrame (Khuyên dùng nhất)
df = spark.sql("""
    SELECT * 
    FROM metastore.lakehouse_db.customer_orders 
    WHERE status = 'COMPLETED'
""")
df.explain(mode="formatted")
```

```
Field 1
== Physical Plan ==
*(1) Filter (isnotnull(status#137) AND (status#137 = COMPLETED))
+- *(1) ColumnarToRow
   +- BatchScan metastore.lakehouse_db.customer_orders[order_id#134L, customer_id#135L, order_amount#136, status#137, payment_method#138, order_date#139, discount_percent#140] metastore.lakehouse_db.customer_orders (branch=null) [filters=status IS NOT NULL, status = 'COMPLETED', groupedBy=] RuntimeFilters: []
```

Kế hoạch thực thi vật lý (Physical Plan) này xác nhận truy vấn đọc dữ liệu Iceberg đã hoạt động hoàn hảo. Dưới đây là phân tích chi tiết từng tầng xử lý từ dưới lên trên:

**1. `BatchScan metastore.lakehouse_db.customer_orders [...]`**

- Spark đang sử dụng **Iceberg BatchScan** (thay vì Parquet scan mặc định của Spark) để tương tác trực tiếp với metadata của Apache Iceberg.
    
- Danh sách các cột đọc lên bao gồm cả `payment_method` và `discount_percent` — chứng minh các thao tác thêm cột (**Schema Evolution**) trước đó đã được cập nhật chuẩn xác vào metadata trên Hive Metastore/GCS.
    

**2. `[filters=status IS NOT NULL, status = 'COMPLETED']` (Predicate Pushdown)**

- Catalyst Optimizer đã đẩy toàn bộ điều kiện `WHERE` xuống tầng đọc metadata của Iceberg.
    
- Trước khi Worker đọc bất kỳ file dữ liệu nào từ GCS, Iceberg sẽ kiểm tra metadata thống kê (min/max bounds, null counts) trong file manifest. File Parquet nào không chứa giá trị `COMPLETED` sẽ bị loại bỏ ngay lập tức (**Data Skipping**), giúp tiết kiệm băng thông và giảm đáng kể thời gian I/O trên Google Cloud Storage.
    

**3. `*(1) ColumnarToRow` & `*(1) Filter`**

- **`ColumnarToRow`**: Dữ liệu từ file Parquet dạng cột (columnar format) được giải nén và chuyển đổi sang dạng hàng (row format) trong bộ nhớ RAM để đưa vào xử lý.
    
- **`Filter`**: Bộ lọc bảo đảm lần hai trong bộ nhớ JVM của Worker để loại bỏ triệt để các bản ghi không khớp trong các file Parquet đã nạp.
    
- Ký hiệu **`*(1)`** (Whole-Stage Code Generation): Spark tổng hợp các bước đọc, chuyển đổi và lọc vào trong một hàm Java bytecode duy nhất, tránh chi phí chuyển đổi dữ liệu trung gian và tối ưu bộ nhớ cache của CPU.


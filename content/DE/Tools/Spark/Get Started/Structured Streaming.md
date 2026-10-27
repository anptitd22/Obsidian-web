## 1. OVERVIEW

- **Structured Streaming:** Engine xử lý dữ liệu dạng luồng (streaming) có khả năng mở rộng (scalable) và chịu lỗi (fault-tolerant), xây dựng trên nền tảng **Spark SQL**.
    
- **Đặc điểm cốt lõi:**
    
    - **Unified API:** Sử dụng chung API Dataset/DataFrame với xử lý theo lô (batch). Viết truy vấn như trên dữ liệu tĩnh, Spark SQL sẽ tự động chạy tính toán tăng dần (incremental).
        
    - **Chịu lỗi Exactly-Once:** Đảm bảo chính xác một lần từ đầu đến cuối (End-to-End Exactly-Once) nhờ **Checkpointing** và **Write-Ahead Logs (WAL)**.
        
- **2 Execution Engines (Chế độ xử lý):**
    
    1. **Micro-Batch Processing (Mặc định):**
        
        - Xử lý luồng thành các đợt lô nhỏ nối tiếp nhau.
            
        - Độ trễ (Latency): $\approx 100\text{ ms}$.
            
        - Cam kết chịu lỗi: **Exactly-Once**.
            
    2. **Continuous Processing (Từ Spark 2.3+):**
        
        - Xử lý liên tục theo thời gian thực.
            
        - Độ trễ (Latency): siêu thấp, chỉ $\approx 1\text{ ms}$.
            
        - Cam kết chịu lỗi: **At-Least-Once**.

## 2. QUICK EXAMPLE - STREAMING WORD COUNT

### A. Khởi tạo SparkSession

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import explode, split

spark = SparkSession.builder \
    .appName("StructuredNetworkWordCount") \
    .getOrCreate()
```

### B. Khởi tạo Streaming DataFrame (Input Stream)

```python
# Tạo Stream DataFrame từ socket kết nối tới localhost:9999
lines = spark.readStream \
    .format("socket") \
    .option("host", "localhost") \
    .option("port", 9999) \
    .load()
```

### C. Biến đổi dữ liệu (Transformations)

```python
# Tách dòng thành các từ (tương tự như trên DataFrame tĩnh)
words = lines.select(
    explode(split(lines.value, " ")).alias("word")
)

# Đếm số lần xuất hiện của từng từ (Aggregations)
wordCounts = words.groupBy("word").count()
```

### D. Ghi đầu ra và khởi chạy Stream (Output Sink)

```python
# Bắt đầu chạy truy vấn, in kết quả đầy đủ ra Console
query = wordCounts.writeStream \
    .outputMode("complete") \
    .format("console") \
    .start()

# Giữ tiến trình chạy ngầm cho đến khi bị ngắt
query.awaitTermination()
```

### E. Thao tác chạy demo qua Terminal

**Terminal 1 (Mở Socket Data Server):**

```bash
$ nc -lk 9999
```

**Terminal 2 (Gửi ứng dụng Spark):**

```bash
$ ./bin/spark-submit streaming_wordcount.py localhost 9999

-------------------------------------------
Batch: 0
-------------------------------------------
+------+-----+
| value|count|
+------+-----+
|apache|    1|
| spark|    1|
+------+-----+

-------------------------------------------
Batch: 1
-------------------------------------------
+------+-----+
| value|count|
+------+-----+
|apache|    2|
| spark|    1|
|hadoop|    1|
+------+-----+
...
```

## 3. PROGRAMMING MODEL

### A. Khái niệm "Unbounded Table"


![[spark_structured_streaming_1.png]]


- Coi luồng dữ liệu đầu vào là một **Input Table** không giới hạn (liên tục có hàng mới được thêm vào khi dữ liệu mới tới).
    
- **Result Table:** Bảng kết quả được truy vấn từ Input Table. Cứ sau mỗi khoảng thời gian kích hoạt (_Trigger Interval_), dữ liệu mới sẽ cập nhật vào Result Table.


![[spark_structured_streaming_2.png]]

### B. Các chế độ đầu ra (Output Modes)

|**Output Mode**|**Cách hoạt động**|**Trường hợp áp dụng**|
|---|---|---|
|**Complete Mode**|Ghi **toàn bộ** Result Table đã được cập nhật ra bộ lưu trữ bên ngoài (External Storage).|Phù hợp cho các truy vấn có gom nhóm/tổng hợp (**Aggregations**).|
|**Append Mode**|Chỉ ghi những **hàng mới được thêm vào** Result Table kể từ lần Trigger cuối.|Áp dụng cho truy vấn mà các hàng cũ **không bao giờ bị thay đổi** (VD: `select`, `filter`).|
|**Update Mode**|Chỉ ghi các **hàng bị thay đổi/cập nhật** trong Result Table kể từ lần Trigger cuối (Hỗ trợ từ Spark 2.1.1).|Phù hợp khi chỉ muốn đẩy phần delta đã thay đổi. Nếu truy vấn không có aggregation, nó tương đương với Append Mode.|


![[spark_structured_streaming_3.png]]

## 4. LƯU Ý QUAN TRỌNG VỀ BỘ NHỚ & TRẠNG THÁI

**Cơ chế lưu trữ trạng thái tối thiểu:**

- Structured Streaming **KHÔNG** lưu toàn bộ dữ liệu lịch sử trong bộ nhớ.
    
- Nó chỉ đọc dữ liệu mới nhất từ Streaming Data Source, xử lý tăng dần (**Incremental Processing**), ghi kết quả ra ngoài rồi **hủy ngay dữ liệu gốc (source data)**.
    
- Hệ thống chỉ giữ lại dữ liệu trạng thái trung gian tối thiểu (_Intermediate State_) bắt buộc phải có để phục vụ cập nhật kết quả (ví dụ: các biến đếm trung gian).


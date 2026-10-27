**Bản chất:** Cổng giao tiếp chuẩn hiện đại (từ Spark 2.0+), gom toàn bộ các context rời rạc (`SparkContext`, `SQLContext`, `HiveContext`) vào một nơi duy nhất.

**Mục đích sử dụng:**

- Làm việc với các API cấp cao: **DataFrame**, **Dataset**, **Spark SQL**.
    
- Lựa chọn mặc định cho hầu hết các bài toán xử lý dữ liệu thực tế nhờ tính tiện lợi và khả năng tối ưu hóa (Catalyst Optimizer).

Ví dụ:

```python
python from pyspark.sql import SparkSession

#Create a SparkSession 
spark = SparkSession.builder
.appName("example_spark_session")  
.getOrCreate()
```

- `SparkSession.builder` bắt đầu quá trình xây dựng một `SparkSession`. 

- `appName("example_spark_session")` đặt tên cho ứng dụng của bạn. 

- `getOrCreate()` có thể truy xuất một `SparkSession` hiện có hoặc tạo một `SparkSession` mới.

Liên quan:

- [[SparkContext & SparkSession]]




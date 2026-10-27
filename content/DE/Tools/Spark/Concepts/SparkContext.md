**Bản chất:** Cổng giao tiếp nguyên bản (từ Spark 1.x), kết nối trực tiếp application với cluster và cluster manager.
    
**Mục đích sử dụng:**
    
- Thao tác trực tiếp với **RDD (Resilient Distributed Datasets)**.
        
- Cần can thiệp vào các tác vụ, cấu hình hoặc cơ chế điều phối cấp thấp (low-level operations).

Ví dụ:

```python
python from pyspark import SparkContext, SparkConf

#Create a Spark configuration 
conf = SparkConf().setAppName("example_spark_context")

#Create a SparkContext 
sc = SparkContext(conf=conf)
```

- `SparkConf().setAppName("example_spark_context")` thiết lập tên ứng dụng trong cấu hình Spark.

- `SparkContext(conf=conf)` khởi tạo một `SparkContext` với cấu hình đã cho.

Liên quan:

- [[SparkContext & SparkSession]]


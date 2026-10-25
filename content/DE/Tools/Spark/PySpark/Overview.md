PySpark là API Python dành cho Apache Spark. Nó cho phép bạn thực hiện xử lý dữ liệu quy mô lớn, thời gian thực trong môi trường phân tán bằng Python. Nó cũng cung cấp một giao diện dòng lệnh PySpark để phân tích dữ liệu một cách tương tác.

PySpark hỗ trợ tất cả các tính năng của Spark như Spark SQL, DataFrames, Structured Streaming, Machine Learning (MLlib), Pipelines và Spark Core.

![[pyspark_overview_tutorial.png]]

Khởi tạo context thông qua [[Py4J]]

- **Cổng:** Sử dụng một **cổng TCP ngẫu nhiên (ephemeral port)** trên `localhost` (thường nằm trong dải cổng tự do từ 1024 trở lên do hệ điều hành cấp phát động).

**Python Spark Connect Client**

Spark Connect là một kiến ​​trúc máy khách-máy chủ trong Apache Spark cho phép kết nối từ xa đến các cụm Spark từ bất kỳ ứng dụng nào. PySpark cung cấp máy khách cho máy chủ Spark Connect, cho phép sử dụng Spark như một dịch vụ.

- [Quickstart: Spark Connect](https://spark.apache.org/docs/latest/api/python/getting_started/quickstart_connect.html)
- [Live Notebook: Spark Connect](https://mybinder.org/v2/gh/apache/spark/32f72996011?filepath=python%2Fdocs%2Fsource%2Fgetting_started%2Fquickstart_connect.ipynb)
- [Spark Connect Overview](https://spark.apache.org/docs/latest/spark-connect-overview.html)
- [Eager vs Lazy: Spark Connect vs Spark Classic](https://spark.apache.org/docs/latest/spark-connect-gotchas.html)

**Spark SQL and DataFrames**

Spark SQL là mô-đun của Apache Spark dùng để làm việc với dữ liệu có cấu trúc. Nó cho phép bạn kết hợp liền mạch các truy vấn SQL với các chương trình Spark. Với PySpark DataFrames, bạn có thể đọc, ghi, chuyển đổi và phân tích dữ liệu một cách hiệu quả bằng Python và SQL. Cho dù bạn sử dụng Python hay SQL, cùng một công cụ thực thi cơ bản được sử dụng, vì vậy bạn luôn tận dụng được toàn bộ sức mạnh của Spark.

- [Quickstart: DataFrame](https://spark.apache.org/docs/latest/api/python/getting_started/quickstart_df.html)
- [Live Notebook: DataFrame](https://mybinder.org/v2/gh/apache/spark/32f72996011?filepath=python%2Fdocs%2Fsource%2Fgetting_started%2Fquickstart_df.ipynb)
- [Spark SQL API Reference](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/index.html)

**Pandas API on Spark**

API Pandas trên Spark cho phép bạn mở rộng quy mô khối lượng công việc Pandas của mình đến bất kỳ kích thước nào bằng cách chạy nó phân tán trên nhiều node. Nếu bạn đã quen thuộc với Pandas và muốn tận dụng Spark cho dữ liệu lớn, API Pandas trên Spark sẽ giúp bạn làm việc hiệu quả ngay lập tức và cho phép bạn di chuyển ứng dụng của mình mà không cần sửa đổi mã. Bạn có thể có một cơ sở mã duy nhất hoạt động cả với Pandas (tests, smaller datasets) và với Spark (production, distributed datasets) và bạn có thể chuyển đổi giữa API Pandas và API Pandas trên Spark một cách dễ dàng và không tốn thêm chi phí.

API Pandas trên Spark nhằm mục đích giúp quá trình chuyển đổi từ pandas sang Spark trở nên dễ dàng, nhưng nếu bạn mới làm quen với Spark hoặc đang phân vân nên sử dụng API nào, chúng tôi khuyên bạn nên sử dụng PySpark (xem [Spark SQL and DataFrames](https://spark.apache.org/docs/latest/api/python/index.html#index-page-spark-sql-and-dataframes)).

- [Quickstart: Pandas API on Spark](https://spark.apache.org/docs/latest/api/python/getting_started/quickstart_ps.html)
- [Live Notebook: Pandas API on Spark](https://mybinder.org/v2/gh/apache/spark/32f72996011?filepath=python%2Fdocs%2Fsource%2Fgetting_started%2Fquickstart_ps.ipynb)
- [Pandas API on Spark Reference](https://spark.apache.org/docs/latest/api/python/reference/pyspark.pandas/index.html)

**Structured Streaming**

Structured Streaming là một công cụ xử lý luồng dữ liệu có khả năng mở rộng và chịu lỗi, được xây dựng trên nền tảng Spark SQL. Bạn có thể thể hiện quá trình tính toán luồng dữ liệu của mình theo cùng một cách như khi thực hiện tính toán theo batch trên dữ liệu tĩnh. Công cụ Spark SQL sẽ đảm bảo việc chạy dữ liệu một cách incrementally and continuously, đồng thời cập nhật kết quả cuối cùng khi dữ liệu luồng tiếp tục được truyền đến.

- [Structured Streaming Programming Guide](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [Structured Streaming API Reference](https://spark.apache.org/docs/latest/api/python/reference/pyspark.ss/index.html)

**Machine Learning (MLlib)**

Được xây dựng trên nền tảng Spark, MLlib là một thư viện học máy có khả năng mở rộng, cung cấp một high-level APIs thống nhất giúp người dùng tạo và tinh chỉnh các machine learning pipelines.

- [Machine Learning Library (MLlib) Programming Guide](https://spark.apache.org/docs/latest/ml-guide.html)
- [Machine Learning (MLlib) API Reference](https://spark.apache.org/docs/latest/api/python/reference/pyspark.ml.html)

**Declarative Pipelines**

Spark Declarative Pipelines (SDP) là một framework khai báo để xây dựng các pipeline dữ liệu đáng tin cậy, dễ bảo trì và dễ kiểm thử trên Spark. SDP đơn giản hóa quá trình phát triển ETL bằng cách cho phép bạn tập trung vào các phép biến đổi mà bạn muốn áp dụng cho dữ liệu của mình, thay vì các cơ chế thực thi pipeline.

- [Pipelines API Reference](https://spark.apache.org/docs/latest/api/python/reference/pyspark.pipelines.html)

**Spark Core and RDDs**

Spark Core là công cụ thực thi tổng quát cơ bản cho nền tảng Spark, là nền tảng cho tất cả các chức năng khác. Nó cung cấp RDD (Resilient Distributed Datasets) và khả năng tính toán trong bộ nhớ.

Lưu ý rằng API RDD là low-level API, có thể khó sử dụng và bạn sẽ không tận dụng được khả năng tối ưu hóa truy vấn tự động của Spark. Chúng tôi khuyên bạn nên sử dụng DataFrame (xem [Spark SQL and DataFrames](https://spark.apache.org/docs/latest/api/python/index.html#index-page-spark-sql-and-dataframes) ở trên) thay vì RDD vì nó cho phép bạn thể hiện điều mình muốn dễ dàng hơn và cho phép Spark tự động xây dựng truy vấn hiệu quả nhất cho bạn.

- [Spark Core API Reference](https://spark.apache.org/docs/latest/api/python/reference/pyspark.html)

**Spark Streaming (Legacy)**

Spark Streaming là một phần mở rộng của API cốt lõi của Spark, cho phép xử lý luồng dữ liệu trực tiếp với khả năng mở rộng, thông lượng cao và khả năng chịu lỗi.

Lưu ý rằng Spark Streaming là thế hệ trước của Spark’s streaming engine. Đây là một dự án cũ và không còn được cập nhật nữa. Hiện có một công cụ xử lý dữ liệu luồng mới hơn và dễ sử dụng hơn trong Spark gọi là Structured Streaming mà bạn nên sử dụng cho các ứng dụng và quy trình xử lý dữ liệu luồng của mình.

- [Spark Streaming Programming Guide (Legacy)](https://spark.apache.org/docs/latest/streaming-programming-guide.html)
- [Spark Streaming API Reference (Legacy)](https://spark.apache.org/docs/latest/api/python/reference/pyspark.streaming.html)


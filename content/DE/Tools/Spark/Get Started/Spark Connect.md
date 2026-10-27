## Core Architecture

**Kiến trúc Client-Server tách biệt (Decoupled Architecture)**: Giới thiệu từ Spark 3.4, Spark Connect cho phép các ứng dụng kết nối từ xa đến cụm Spark thông qua DataFrame API và giao thức kế hoạch logic chưa phân giải (unresolved logical plans).

![[Pasted image 20261027014819.png]]

**Luồng xử lý (How it works)**:

![[Pasted image 20261027015151.png]]

1. **Tạo kế hoạch ở Client**: Thư viện client dịch các thao tác DataFrame thành kế hoạch truy vấn logic chưa phân giải, mã hóa bằng **Protocol Buffers**.
2. **Truyền tải**: Kế hoạch được gửi tới Spark Server thông qua khung **gRPC**.
3. **Thực thi ở Server**: Spark Server chuyển đổi kế hoạch này thành toán tử kế hoạch logic, áp dụng đầy đủ các cơ chế tối ưu hóa của Spark.
4. **Trả kết quả**: Kết quả được phát trực tuyến (streamed) về client qua gRPC dưới dạng các lô dòng mã hóa **Apache Arrow**.

>[!note]
>Khi chạy spark connect sẽ tạo 1 spark context xuyên suốt và nhiều người dùng dùng chung spark context này nếu kết nối thông qua 1 cổng spark connect

## Lợi ích vận hành (Operational Benefits)

- **Tính ổn định (Stability)**: Ứng dụng quá tải bộ nhớ chỉ ảnh hưởng đến tiến trình của chính nó; các thư viện phụ thuộc ở phía client không gây xung đột với Spark Driver.

- **Nâng cấp độc lập (Upgradability)**: Spark Driver có thể nâng cấp độc lập (vá lỗi, tối ưu hiệu năng) mà không bắt buộc ứng dụng phía client phải sửa đổi hay biên dịch lại.

- **Khả năng quan sát & Gỡ lỗi (Debuggability & Observability)**: Hỗ trợ gỡ lỗi tương tác (interactive debugging) trực tiếp từ IDE và theo dõi thông qua các thư viện logging/metrics bản địa.

- **Xác thực (Authentication)**: Không tích hợp sẵn logic xác thực nội bộ nhưng giao diện gRPC HTTP/2 tương thích tốt với các proxy xác thực bên ngoài.

## Hỗ trợ API (Supported APIs)

- **Được hỗ trợ**:
    - **PySpark**: Hỗ trợ hầu hết các API `DataFrame`, `Functions`, và `Column`.
    - **Scala**: Hỗ trợ các API `Dataset`, `functions`, `Column`, `Catalog`, `KeyValueGroupedDataset` và hầu hết Streaming API (`DataStreamReader`, `DataStreamWriter`, v.v.).

- **Không hỗ trợ / Đã lỗi thời (Deprecated)**:
    - Các API cấp thấp như [[SparkContext]] và **RDD** **không được hỗ trợ** trong PySpark và bị đánh dấu **deprecated** trong Scala khi dùng Spark Connect.

## Cấu hình & Sử dụng cơ bản (Usage & Setup)

- **Khởi chạy Spark Server**:
    - Chạy lệnh: `./sbin/start-connect-server.sh --packages org.apache.spark:spark-connect_2.12:3.5.9`.
- **Kết nối qua PySpark Shell**:
    - Thiết lập môi trường: `export SPARK_REMOTE="sc://localhost"` hoặc khởi chạy với cờ `--remote`: `./bin/pyspark --remote "sc://localhost"`.
- **Dùng trong Ứng dụng Độc lập (Standalone App)**:
    - **Python**: Cài đặt gói `pyspark[connect]==3.5.9` và khởi tạo session qua `SparkSession.builder.remote("sc://localhost").getOrCreate()`.
    - **Scala**: Khai báo phụ thuộc `spark-connect-client-jvm` và khởi tạo session qua `SparkSession.builder().remote("sc://localhost").build()`.

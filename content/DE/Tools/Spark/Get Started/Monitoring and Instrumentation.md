## 1. Khái niệm chung (Overview)

- Spark cung cấp 3 phương thức chính để giám sát ứng dụng: [[Spark UI]], **Hệ thống Metrics**, và các **Công cụ đo đạc bên ngoài (External Instrumentation)**.

## 2. Giao diện Web (Web Interfaces)

### A. Web UI trực tiếp (Live Web UI)

- Mọi `SparkContext` khi khởi chạy đều tự động mở một Web UI tại cổng mặc định **4040** (`http://<driver-node>:4040`). Nếu có nhiều SparkContext cùng chạy trên một host, cổng sẽ tự động tăng dần (4041, 4042,...).
- **Thành phần hiển thị**: Danh sách các scheduler stages và tasks, kích thước RDD cùng mức sử dụng bộ nhớ, thông tin môi trường hệ thống, và danh sách các executor đang hoạt động.
- Mặc định, thông tin Web UI trực tiếp này chỉ tồn tại trong suốt thời gian ứng dụng đang chạy.

### B. Xem lại nhật ký sau khi chạy (Spark History Server)

- **Kích hoạt ghi log**: Cần bật cấu hình `spark.eventLog.enabled = true` và khai báo đường dẫn thư mục lưu trữ nhật ký qua `spark.eventLog.dir`.

- **Khởi chạy History Server**: Thực thi lệnh `./sbin/start-history-server.sh` để mở giao diện Web tại cổng mặc định **18080** (`http://<server-url>:18080`) giúp xem lại lịch sử các ứng dụng.

- **Nén file nhật ký (Event Log Compaction)**:
    - Khi bật ghi log cuộn (`spark.eventLog.rolling.enabled`), History Server có thể nén file nhật ký thông qua thuộc tính `spark.history.fs.eventLog.rolling.maxFilesToRetain`.
    - Đây là thao tác **có mất mát dữ liệu (lossy operation)**, giúp giảm dung lượng bằng cách loại bỏ các sự kiện đã kết thúc hoặc không còn hiệu lực (như job, stage hay executor đã dừng).

- **Các cấu hình quan trọng khác**:
    - `spark.history.fs.update.interval` _(Mặc định: 10s)_: Tần suất quét định kỳ để cập nhật file log mới.
    - `spark.history.retainedApplications` _(Mặc định: 50)_: Số lượng ứng dụng được giữ dữ liệu UI trong cache.
    - `spark.history.store.serializer`: Định dạng chuỗi hóa lưu cache UI (`JSON` hoặc `PROTOBUF` từ Spark 3.4.0).

## 3. REST API

- Tất cả dữ liệu đo đạc hiển thị trên UI đều có thể truy xuất dưới dạng JSON thông qua REST API tại đường dẫn gốc `/api/v1` (ví dụ: `http://localhost:4040/api/v1` hoặc `http://<server-url>:18080/api/v1`).

- **Các Endpoint chính**:
    - `/applications`: Danh sách các ứng dụng (có thể lọc theo trạng thái `status` hoặc khoảng thời gian).
    - `/applications/[app-id]/jobs` & `/stages`: Thông tin chi tiết về từng job, stage và phân bố task metrics.
    - `/applications/[app-id]/executors`: Danh sách các executor đang hoạt động.
    - `/applications/[app-id]/sql` & `/storage/rdd`: Thông tin chi tiết các truy vấn SQL và trạng thái lưu trữ RDD.

- **Chỉ số công việc của Executor (Executor Task Metrics)**: Cung cấp chi tiết mức độ xử lý ở cấp độ task như `executorRunTime`, `executorCpuTime`, `jvmGCTime`, `memoryBytesSpilled`, `diskBytesSpilled`, `inputMetrics`, `outputMetrics`, `shuffleReadMetrics`, `shuffleWriteMetrics`.

- **Chỉ số bộ nhớ Executor (Executor Metrics)**: Được gửi về Driver thông qua heartbeat, hỗ trợ xuất định dạng Prometheus tại `/metrics/executors/prometheus` khi bật `spark.ui.prometheus.enabled=true`.

## 4. Hệ thống Metrics (Metrics System)

- Xây dựng dựa trên thư viện **Dropwizard Metrics Library**.

- **Cấu hình**: Thiết lập qua tệp `$SPARK_HOME/conf/metrics.properties` hoặc tiền tố cấu hình `spark.metrics.conf.`. Có thể đặt không gian tên tùy chỉnh qua `spark.metrics.namespace`.

- **Hỗ trợ các Instance (Thành phần)**: `driver`, `executor`, `master`, `worker`, `applications`, `shuffleService`, `applicationMaster`, `mesos_cluster`.

- **Các Sink (Đầu ra dữ liệu)**:
    - Mặc định hỗ trợ: `ConsoleSink`, `CSVSink`, `JmxSink`, `MetricsServlet` (trả về JSON tại `/metrics/json`), `PrometheusServlet`, `GraphiteSink`, `Slf4jSink`, `StatsdSink`.
    - `GangliaSink`: Cần tự biên dịch riêng Spark với cờ LGPL do chính sách bản quyền.

## 5. Đo đạc nâng cao & Plugin API (Advanced Instrumentation)

- **Công cụ giám sát hệ thống**:
    - Giám sát toàn cụm: Ganglia.
    - Giám sát hệ điều hành từng nút: `dstat`, `iostat`, `iotop`.
    - Công cụ JVM: `jstack` (xem stack trace), `jmap` (tạo heap dump), `jstat`, `jconsole`.

- **Plugin API**: Cho phép nạp mã đo đạc tùy chỉnh vào Spark bằng cách triển khai giao diện `org.apache.spark.api.plugin.SparkPlugin` và khai báo qua thuộc tính `spark.plugins` hoặc `spark.plugins.defaultList`.


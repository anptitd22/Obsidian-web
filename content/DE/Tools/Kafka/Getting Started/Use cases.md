## 1. Messaging (Hệ thống truyền thông điệp)

- **Vai trò:** Thay thế cho các message broker truyền thống như **ActiveMQ** hoặc **RabbitMQ**.

- **Đặc điểm nổi bật:**
    - **Thông lượng cao** (throughput) và **độ trễ thấp**.
    - Hỗ trợ **phân vùng tích hợp** (built-in partitioning), **nhân bản** (replication), và **khả năng chịu lỗi** (fault-tolerance) tốt cho các ứng dụng quy mô lớn.
    - Đảm bảo **độ bền dữ liệu cao** (strong durability guarantees).

---

## 2. Website Activity Tracking (Theo dõi hoạt động trên Website)

- **Mục đích ban đầu:** Đây là use case gốc khi Kafka mới ra đời, dùng để xây dựng lại đường ống theo dõi hoạt động người dùng theo mô hình **Publish-Subscribe**.

- **Cách hoạt động:**
    - Các thao tác (lượt xem trang, tìm kiếm, v.v.) được đẩy vào các **topic trung tâm**.
    - Dữ liệu được đăng ký tiêu thụ để **xử lý/giám sát theo thời gian thực** hoặc nạp vào **Hadoop / Data Warehouse** để xử lý offline.

- **Đặc điểm:** Xử lý **khối lượng dữ liệu cực lớn** (high volume) do mỗi hành động của người dùng sinh ra rất nhiều thông điệp.

---

## 3. Metrics (Thu thập chỉ số & Giám sát hệ thống)

- **Ứng dụng:** Thu thập và tổng hợp dữ liệu thống kê, dữ liệu vận hành từ nhiều ứng dụng phân tán.

- **Kết quả:** Tạo ra các luồng dữ liệu trung tâm phục vụ cho việc **giám sát vận hành** (operational monitoring).

---

## 4. Log Aggregation (Gom nhật ký hệ thống)

- **Thay thế:** Các giải pháp gom log truyền thống dựa trên file (như Scribe hay Flume).

- **Ưu điểm của Kafka:**
    - Trừu tượng hóa dữ liệu log/sự kiện thành **chuỗi thông điệp** (stream of messages).
    - **Độ trễ thấp hơn** và hỗ trợ tiêu thụ dữ liệu phân tán dễ dàng hơn.
    - **Độ bền dữ liệu mạnh mẽ hơn** nhờ cơ chế replication.

---

## 5. Stream Processing (Xử lý dòng dữ liệu)

- **Cách vận hành:** Dữ liệu thô từ topic này được tiêu thụ, **tổng hợp, làm giàu (enrich) hoặc biến đổi**, sau đó ghi sang topic mới cho các bước xử lý tiếp theo.

- **Công cụ hỗ trợ:**
    - Thư viện **Kafka Streams** tích hợp sẵn (từ phiên bản 0.10.0.0).
    - Các công cụ mã nguồn mở khác như **Apache Storm**, **Apache Samza**.

---

## 6. Event Sourcing (Lưu trữ sự kiện)

- **Khái niệm:** Phong cách thiết kế ứng dụng trong đó các thay đổi trạng thái được ghi lại dưới dạng **chuỗi bản ghi sắp xếp theo thời gian**.

- **Lý do dùng Kafka:** Khả năng lưu trữ log dung lượng lớn giúp Kafka trở thành **backend lý tưởng** cho kiến trúc Event Sourcing.

---

## 7. Commit Log (Nhật ký ghi nhận cho hệ thống phân tán)

- **Vai trò:** Đóng vai trò như một **external commit-log** cho các hệ thống phân tán.

- **Tác dụng:**
    - Giúp **đồng bộ/nhân bản dữ liệu** giữa các node.
    - Hỗ trợ **khôi phục dữ liệu** cho các node bị lỗi (dựa trên tính năng _log compaction_, tương tự Apache BookKeeper).
## 1. Đặc điểm chung của Giao thức (Network & Protocol Basics)

- **Giao thức nhị phân qua TCP:** Kafka sử dụng giao thức nhị phân (binary protocol) chạy trên nền TCP, định nghĩa mọi giao tiếp dưới dạng các cặp **Request - Response**1.
- **Định dạng thông điệp:** Các thông điệp đều có giới hạn độ dài (size-delimited) và cấu thành từ các kiểu dữ liệu nguyên thủy (primitive types)1more_horiz.
- **Không cần Handshake:** Client khởi tạo socket connection, gửi chuỗi request và đọc response trực tiếp mà không cần thủ tục bắt tay giao thức ở tầng ứng dụng khi kết nối hay ngắt kết nối1.
- **Kết nối bền vững (Persistent Connection):** Khuyến nghị duy trì kết nối socket lâu dài để giảm chi phí bắt tay TCP cho nhiều yêu cầu liên tiếp1.
- **Giới hạn kích thước:** Server có cấu hình giới hạn kích thước request tối đa; nếu request vượt quá giới hạn này, socket sẽ bị đóng1.

---

## 2. Phân vùng dữ liệu & Khởi tạo kết nối (Partitioning & Bootstrapping)

- **Kiến trúc phân vùng:** Topic chia thành các phân vùng $0, 1, \dots, P-1$ và mỗi phân vùng được nhân bản với hệ số $N$4.
- **Quyền điều khiển của Client:** Client trực tiếp quyết định việc phân bổ dữ liệu vào partition nào khi ghi hoặc đọc4.
- **Gửi tới Leader Broker:** Các request ghi (Produce) hoặc đọc (Fetch) bắt buộc phải gửi tới đúng **Leader broker** của partition đó. Gửi nhầm broker sẽ trả về lỗi `NotLeaderForPartition`5.
- **Cơ chế Bootstrapping & Metadata:**
    - Client kết nối tới danh sách một vài broker ban đầu (bootstrap URLs) để gửi `Metadata Request`5more_horiz.
    - Metadata trả về thông tin toàn cluster: các topic hiện có, các partition, broker leader tương ứng và thông tin IP/Port của các broker5.
    - Client sẽ cache dữ liệu metadata này và chỉ làm mới (refresh) khi gặp lỗi socket hoặc nhận thông báo metadata đã cũ67.

---

## 3. Chiến lược Phân vùng & Gom cụm (Partitioning & Batching)

- **Phân vùng ngữ nghĩa (Semantic Partitioning):** Client sử dụng key của thông điệp kết hợp với hàm băm (hash) để đảm bảo toàn bộ thông điệp cùng key (ví dụ: `user_id`) đi vào cùng một partition8.
- **Gom cụm dữ liệu (Batching):** Cho phép gộp nhiều bản ghi (record batch) thuộc nhiều topic/partition khác nhau vào cùng một request Produce hoặc Fetch để tăng hiệu năng mạng8.

---

## 4. Quản lý Phiên bản & Khả năng Tương thích (API Versioning & Compatibility)

- **Tương thích hai chiều (Bidirectional Compatibility):** Client phiên bản mới có thể giao tiếp với Broker phiên bản cũ và ngược lại9.
- **Cặp API Key & API Version:** Mỗi request bao gồm **API Key** (mã số định danh API) và **API Version** (số 16-bit) để xác định chính xác cấu trúc (schema) của thông điệp10.
- **Truy vấn phiên bản (****ApiVersionsRequest****):** Client gửi yêu cầu này tới broker để lấy dải phiên bản API mà broker hỗ trợ, sau đó chọn phiên bản cao nhất mà cả 2 bên cùng đáp ứng1112.

---

## 5. Cấu trúc Khung thông điệp (Common Structure & Headers)

- **Khung tổng quát:** Bắt đầu bằng 4 byte `Size` (định kích thước thông điệp theo sau), theo sau là Header và Body313.
- **Request Header:** Bao gồm `request_api_key` (INT16), `request_api_version` (INT16), `correlation_id` (INT32), và `client_id` (NULLABLE_STRING)1314.
- **Response Header:** Chứa `correlation_id` (INT32) giúp client khớp chính xác phản hồi trả về với yêu cầu đã gửi15.

---

## 6. Danh mục Mã API Key phổ biến (Common APIs)

- **Produce (Key 0):** Gửi dữ liệu tới partition (chứa các tham số như `acks`, `timeout_ms`, `topic_data`)1516.
- **Fetch (Key 1):** Rút/Đọc dữ liệu từ partition1617.
- **ListOffsets (Key 2):** Truy vấn danh sách vị trí offset theo mốc thời gian1618.
- **Metadata (Key 3):** Lấy thông tin cấu trúc cluster và phân vùng1619.
- **OffsetCommit (Key 8) & OffsetFetch (Key 9):** Lưu trữ và truy vấn offset đã commit của consumer group16more_horiz.
- **FindCoordinator (Key 10), JoinGroup (Key 11), Heartbeat (Key 12):** Quản lý điều phối consumer group16more_horiz.

---

## 7. Triết lý Thiết kế Giao thức (Design Philosophy)

- Kafka không dùng các hệ thống có sẵn như **Protocol Buffers** hay **Thrift** để tự phát triển binary protocol vì cần kiểm soát trực tiếp sự khớp nối giữa định dạng log lưu trên đĩa và giao thức mạng, đồng thời chủ động quản lý việc tương thích phiên bản (versioning)25.

---
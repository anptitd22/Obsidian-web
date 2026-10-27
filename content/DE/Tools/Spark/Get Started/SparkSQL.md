## 1. Khái niệm cốt lõi (Core Concepts)

- **Spark SQL**: Là một module của Spark chuyên dùng cho việc xử lý dữ liệu có cấu trúc (structured data processing).

- **Điểm vượt trội so với RDD API**:
    - Cung cấp cho Spark thông tin chi tiết hơn về **cấu trúc của dữ liệu** và **phép tính toán** đang thực thi.
    - Spark sử dụng thông tin bổ sung này để thực hiện các **cơ chế tối ưu hóa nội bộ**.

- **Bộ máy thực thi thống nhất (Unified Execution Engine)**: Tất cả các API (SQL, DataFrame, Dataset) và ngôn ngữ đều dùng chung một engine tính toán bên dưới, giúp người dùng dễ dàng chuyển đổi qua lại giữa các API.

## 2. Các giao diện tương tác (Interaction Interfaces)

- **Thực thi câu lệnh SQL**: Cho phép chạy trực tiếp các truy vấn SQL hoặc kết nối với cài đặt Hive có sẵn.

- **Trả về kết quả**: Khi gọi từ các ngôn ngữ lập trình, kết quả truy vấn SQL sẽ được trả về dưới dạng `Dataset`/`DataFrame`.

- **Phương thức kết nối**: Tương tác linh hoạt qua giao diện dòng lệnh (CLI) hoặc thông qua cổng **JDBC/ODBC**.

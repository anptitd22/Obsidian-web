## 1. Nguyên tắc cốt lõi (Core Security Principles)

- **Không bật mặc định**: Các tính năng bảo mật (như xác thực) **không được bật mặc định** trong Spark.

- **Môi trường mở**: Khi triển khai cụm trên mạng mở hoặc không tin cậy, bắt buộc phải chủ động cấu hình bảo mật để ngăn chặn các ứng dụng trái phép thực thi.

- **Giới hạn phạm vi**: Spark chỉ bảo mật những thành phần được hỗ trợ theo tài liệu; các tính năng không được tài liệu đề cập đến sẽ không được Spark hỗ trợ.

## 2. Bảo mật giao tiếp giữa các tiến trình (Spark RPC)

- **Xác thực (Authentication)**:
    - **Cơ chế**: Sử dụng khóa bí mật chia sẻ (shared secret) thông qua cấu hình `spark.authenticate = true`.
    - **YARN**: Spark tự động tạo và phân phối khóa bí mật riêng cho từng ứng dụng.
    - **Kubernetes**: Tự động tạo khóa bí mật và truyền qua biến môi trường hoặc tệp tin (`spark.authenticate.secret.file`, `spark.authenticate.secret.driver.file`, `spark.authenticate.secret.executor.file`).
    - **Lưu ý quan trọng**: Server gửi REST (`REST Submission Server`) và `MesosClusterDispatcher` **không hỗ trợ xác thực** (cổng 6066 và 7077), cần giới hạn truy cập mạng bằng tường lửa.

- **Mã hóa (Encryption)**:
    - Khuyến nghị dùng mã hóa dựa trên **AES** (`spark.network.crypto.enabled = true`, yêu cầu phải bật RPC Authentication trước).
    - **Cipher mode**: Mặc định là `AES/CTR/NoPadding`, nhưng khuyến nghị nâng cấp lên **AES/GCM/NoPadding** để có chế độ mã hóa xác thực.
    - **Engine Version**: Sử dụng phiên bản `2` (`spark.network.crypto.authEngineVersion = 2`) để đảm bảo khóa phiên phân bố đều hơn.

## 3. Mã hóa lưu trữ cục bộ (Local Storage Encryption)

- **Đối tượng bảo vệ**: Dữ liệu tạm thời ghi ra đĩa cục bộ, bao gồm tệp shuffle, shuffle spill và các khối dữ liệu (cached data, broadcast variables). _(Không bao gồm dữ liệu đầu ra từ_ _saveAsHadoopFile_ _hay_ _saveAsTable_).

- **Cấu hình**:
    - `spark.io.encryption.enabled = true`: Bật mã hóa I/O đĩa cục bộ.
    - `spark.io.encryption.keySizeBits`: Độ dài khóa (128, 192, hoặc 256 bits; mặc định là 128).
    - `spark.io.encryption.keygen.algorithm`: Thuật toán tạo khóa (mặc định là `HmacSHA1`).

## 4. Bảo mật Giao diện Web (Web UI Security)

- **Xác thực & Phân quyền (Authentication & Authorization)**:
    - **Xác thực**: Sử dụng các bộ lọc javax servlet (`spark.ui.filters`).
    - **ACLs (Danh sách kiểm soát truy cập)**: Bật qua `spark.acls.enable = true`. Phân định rõ quyền xem (`spark.ui.view.acls`) và quyền chỉnh sửa/hủy job (`spark.modify.acls`). Quyền quản trị viên được cấu hình qua `spark.admin.acls`.
    - **Group Mapping**: Xác định danh sách nhóm người dùng qua `spark.user.groups.mapping` (mặc định dùng shell Unix/Linux, không hỗ trợ Windows).

- **Cấu hình SSL/TLS**:
    - Phân cấp cấu hình theo không gian tên (`${ns}`): `spark.ssl` (mặc định toàn hệ thống), `spark.ssl.ui` (Web UI ứng dụng), `spark.ssl.standalone` (Master/Worker UI), `spark.ssl.historyServer` (History Server UI).
    - Mật khẩu KeyStore/TrustStore có thể lưu an toàn qua **Hadoop Credential Providers** (`jceks://...`).

- **HTTP Security Headers**:
    - `spark.ui.xXssProtection`: Chống XSS (mặc định `1; mode=block`).
    - `spark.ui.xContentTypeOptions.enabled`: Mặc định `true` (`nosniff`).
    - `spark.ui.strictTransportSecurity`: Cấu hình HSTS khi chạy SSL/TLS.

## 5. Cấu hình cổng mạng (Network Security Ports)

- **Chỉ dành cho Standalone Mode**: Web UI Master (8080), Web UI Worker (8081), Master Port (7077), REST Port (6066).

- **Tất cả Trình quản lý Cụm (All Cluster Managers)**: Application Web UI (4040), History Server Web UI (18080), Driver Port (ngẫu nhiên), Block Manager Port (ngẫu nhiên).

## 6. Xác thực Kerberos & Delegation Tokens

- **Cơ chế**: Dùng Kerberos để xác thực và lấy Delegation Token làm việc với các dịch vụ Hadoop (HDFS, Hive, HBase).

- **Ứng dụng chạy lâu dài (Long-Running Applications)**:
    - **Cách 1 - Keytab**: Cung cấp `--principal` và `--keytab` để Spark tự động làm mới token vô thời hạn.
    - **Cách 2 - Ticket Cache**: Đặt `spark.kerberos.renewal.credentials = ccache` để làm mới ticket Kerberos cục bộ trong suốt khoảng thời gian còn hiệu lực.

- **Kubernetes**: Lưu trữ Delegation Tokens dưới dạng Kubernetes Secrets được chia sẻ giữa Driver và Executor.

## 7. Bảo mật tệp nhật ký (Event Logging & Driver Logs)

- Thư mục chứa log sự kiện (`spark.eventLog.dir`) và log driver (`spark.driver.log.dfsDir`) cần được khởi tạo thủ công với quyền **drwxrwxrwxt** (sticky bit) thuộc sở hữu của người dùng chạy Spark History Server.

- Điều này đảm bảo mọi người dùng đều có thể ghi log nhưng không thể đọc, xóa hoặc đổi tên tệp của người khác.
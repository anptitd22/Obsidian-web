---
title: Log
tags:
  - MySQL
---
## Binlog

### Bản chất & Phạm vi

Lưu lại các sự kiện làm thay đổi dữ liệu hoặc cấu trúc DB (DDL, DML) kèm thời gian thực thi.

**Không ghi:** Các câu lệnh chỉ đọc (`SELECT`, `SHOW`).

**Hai mục đích cốt lõi:**

1. **Replication (Sao chép):** Gửi log sang Replicas để đồng bộ dữ liệu.
    
2. **Data Recovery (Khôi phục):** Chạy lại các sự kiện từ sau mốc backup gần nhất (Point-in-Time Recovery).

### Định dạng & Quản lý file

- **3 định dạng ghi (Logging Formats):**
    
    - `ROW`: (Mặc định) Ghi chi tiết từng hàng bị đổi (chính xác nhất, an toàn cho replicate).
        
    - `STATEMENT`: Ghi lại nguyên văn câu lệnh SQL.
        
    - `MIXED`: Tự động chuyển đổi giữa Statement và Row tùy trường hợp.
        
- **Cấu trúc file:** Gồm các file log tuần tự (`binlog.000001`,...) và 1 file chỉ mục (`.index`).
    
- **Quy tắc tạo file mới:** Khởi động lại server, gọi lệnh `FLUSH LOGS`, hoặc file chạm ngưỡng `max_binlog_size`.

- **Lưu ý giao dịch lớn:** Một transaction không bao giờ bị cắt đôi giữa 2 file log; do đó file binlog có thể tạm thời vượt quá `max_binlog_size`.

### Cơ chế xử lý Transaction

- **Thời điểm ghi:** Ngay sau khi lệnh/transaction kết thúc, nhưng **trước khi nhả lock và commit** hoàn toàn.

### An toàn dữ liệu & Xử lý sự cố (Crash Recovery)

- **`binlog_error_action = ABORT_SERVER` (mặc định):** Nếu không thể ghi binlog (ví dụ đầy ổ cứng), MySQL lập tức dừng server để tránh lệch dữ liệu.

### Vận hành & Bảo mật

- **Bảo mật:** Tự động che mật khẩu (ẩn plain text); hỗ trợ mã hóa file log qua `binlog_encryption = ON`.
    
- **Đọc log:** Dùng công cụ dòng lệnh `mysqlbinlog`.
    
- **Quy tắc xóa log an toàn:** Tuyệt đối không xóa tay (`rm`). Phải dùng lệnh SQL `PURGE BINARY LOGS ...` để hệ thống tự động dọn dẹp và cập nhật file `.index`.

## Undo log

Là nhật ký lưu lại trạng thái dữ liệu _trước khi bị thay đổi_. Nếu một transaction thực hiện sửa đổi (Insert, Update, Delete), bản ghi về cách hoàn tác thay đổi đó sẽ được ghi vào Undo Log.

- **Vai trò chính:**
    
    - **Rollback (Đảm bảo Atomicity - Tính nguyên tử):** Khi một transaction gặp lỗi hoặc người dùng gọi lệnh `ROLLBACK`, MySQL dùng Undo Log để khôi phục dữ liệu về đúng trạng thái ban đầu.
        
    - **MVCC - Multi-Version Concurrency Control (Đảm bảo Isolation - Tính cô lập):** Khi một transaction đang đọc dữ liệu trong khi transaction khác đang sửa, MySQL dùng Undo Log để dựng lại phiên bản dữ liệu cũ (snapshot) cho transaction đọc, giúp các tiến trình đọc không bị khóa bởi các tiến trình ghi (Non-blocking reads).
        
- **Cách hoạt động:**
    
    - Với lệnh `INSERT`: Undo Log lưu thông tin để thực hiện `DELETE` bản ghi đó khi cần.
        
    - Với lệnh `DELETE`: Undo Log lưu lại toàn bộ bản ghi để có thể `INSERT` lại.
        
    - Với lệnh `UPDATE`: Undo Log lưu giá trị cũ của các cột vừa bị cập nhật.

## Redo log

Là nhật ký lưu lại các thay đổi _vật lý_ trên các trang dữ liệu (data pages). Đây là nhật ký ghi trước (Write-Ahead Logging - WAL) được tối ưu hóa để ghi tuần tự cực nhanh xuống đĩa cứng.

- **Vai trò chính:**
    
    - **Crash Recovery (Đảm bảo Durability - Tính bền vững):** Khi một transaction `COMMIT`, thay vì tốn chi phí ghi ngẫu nhiên (random I/O) toàn bộ các trang dữ liệu từ bộ nhớ đệm (Buffer Pool) xuống đĩa ngay lập tức, MySQL chỉ cần ghi tuần tự (sequential I/O) thao tác đó vào Redo Log và xác nhận commit thành công. Nếu máy chủ bị sập nguồn đột ngột, khi khởi động lại, MySQL sẽ đọc Redo Log để ghi lại (replay/redo) các thay đổi chưa kịp đưa xuống đĩa dữ liệu chính.
        
- **Cách hoạt động:**
    
    - Gồm hai file xoay vòng (mặc định là `ib_logfile0`, `ib_logfile1` hoặc cấu hình động từ MySQL 8.0.30 trở đi).
        
    - Ghi theo dạng vòng tròn (circular buffer): khi ghi đầy dung lượng được cấp phát, tiến trình nền (_checkpointing_) sẽ đẩy dữ liệu từ Buffer Pool xuống đĩa rồi xoay vòng ghi đè tiếp.


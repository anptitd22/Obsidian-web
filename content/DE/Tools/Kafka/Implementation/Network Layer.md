### 1. Kiến trúc tổng quan (Architecture Overview)

- **Bản chất:** Dựa trên máy chủ **Java NIO** (_Non-blocking I/O_) đơn giản và tốc độ cao.
    
- **Giao thức (Protocol):** Giữ ở mức tinh gọn, đơn giản để thuận tiện cho việc phát triển các bộ thư viện client bằng nhiều ngôn ngữ lập trình khác nhau trong tương lai.
    

### 2. Tối ưu truyền tải dữ liệu (Sendfile & Zero-Copy)

- **Cơ chế triển khai:** Giao diện `TransferableRecords` cung cấp phương thức `writeTo`.
    
- **Hiệu năng:** Cho phép các tệp lưu trữ thông điệp sử dụng trực tiếp cơ chế `transferTo` (gọi lệnh hệ thống `sendfile` trên Linux) thay vì phải ghi đệm (_buffered write_) qua user-space, giúp tối đa hóa tốc độ truyền tải qua mạng.
    

### 3. Mô hình luồng mạng (Threading Model)

- **1 Luồng chấp nhận (`Acceptor Thread`):** Chuyên trách lắng nghe và tiếp nhận các kết nối mạng mới.
    
- **N Luồng xử lý (`Processor/Network Threads`):** Mỗi luồng phụ trách xử lý I/O cho một số lượng kết nối cố định.
    
- **Đặc điểm thiết kế:** Đơn giản trong triển khai, đã được kiểm chứng rộng rãi và đạt hiệu năng thực thi cao.
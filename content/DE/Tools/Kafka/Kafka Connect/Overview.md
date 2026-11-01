## 1. Định nghĩa & Mục đích Cốt lõi

- **Bản chất:** Công cụ chuẩn hóa để tích hợp và truyền phát dữ liệu (_data streaming_) quy mô lớn, tin cậy giữa Apache Kafka và các hệ thống bên ngoài.
    
- **Mục tiêu:** Đơn giản hóa việc định nghĩa các connector để dịch chuyển khối lượng dữ liệu khổng lồ ra/vào Kafka mà không cần tự viết mã nguồn tích hợp từ đầu.
    

## 2. Chiều dịch chuyển dữ liệu (Data Flow)

- **Ingest vào Kafka (Source):** Thu thập toàn bộ cơ sở dữ liệu (CDC) hoặc gom metrics/logs từ application servers vào Kafka topics với độ trễ thấp để phục vụ stream processing.
    
- **Export ra ngoài (Sink):** Đẩy dữ liệu từ Kafka topics sang các hệ thống lưu trữ thứ cấp (secondary storage), hệ thống truy vấn, hoặc các batch systems (Hadoop, Data Lakehouse/Warehouse) để phân tích ngoại tuyến (_offline analysis_).
    

## 3. Các tính năng & Ưu điểm cốt lõi

- **Khung chuẩn hóa chung (Common Framework):**
    
    - Chuẩn hóa cách tích hợp giữa Kafka và hệ thống ngoài.
        
    - Giúp đơn giản hóa quy trình phát triển, triển khai và quản trị connector.
        
- **Chế độ triển khai linh hoạt (Standalone & Distributed Modes):**
    
    - _Standalone:_ Thu gọn cho môi trường dev, test hoặc hệ thống nhỏ gọn.
        
    - _Distributed:_ Cụm phân tán mở rộng, quản lý tập trung cho toàn bộ doanh nghiệp.
        
- **Giao diện quản trị REST API:**
    
    - Khởi tạo, cấu hình, kiểm tra trạng thái và quản lý vòng đời connector dễ dàng qua HTTP requests.
        
- **Quản lý Offset tự động (Automatic Offset Management):**
    
    - Tự động quản lý tiến trình commit offset/checkpoint dữ liệu với hệ thống ngoài.
        
    - Lập trình viên viết connector không cần tự xử lý logic commit vốn phức tạp và dễ phát sinh lỗi.
        
- **Khả năng mở rộng mặc định (Distributed & Scalable by Default):**
    
    - Tận dụng chính giao thức quản lý nhóm (_group management protocol_) sẵn có của Kafka.
        
    - Cho phép thêm/bớt worker linh hoạt để scale-up/scale-down cụm cluster.
        
- **Cầu nối Stream & Batch (Streaming/Batch Integration):**
    
    - Kết nối liền mạch giữa các hệ thống thời gian thực (streaming) và các hệ thống xử lý theo lô (batch/data warehouse).
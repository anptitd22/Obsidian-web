---
title: Architecture
tags:
  - PostgreSQL
  - RDBMS
---

## POSTGRESQL CORE ARCHITECTURE

![[Pasted image 20261024123204.png]]

### Mô Hình Tổng Quan (Process-Based Architecture)

- **Kiến trúc cốt lõi**: Mô hình **Tiến trình-trên-Kết nối (Process-per-connection / Client-Server)**, không sử dụng kiến trúc đa luồng (multi-threaded) cho worker chính.
    
- Mỗi phiên làm việc của máy khách (client session) tương ứng với một tiến trình máy chủ độc lập (Backend Process)

### Các Thành Phần Chính (Core Components)

#### Tiến trình Điều phối Trung tâm (Postmaster / Server Process)

Là tiến trình cha quản lý toàn bộ vòng đời của cơ sở dữ liệu:

- **Khởi động & Tắt hệ thống (Init & Shutdown)**: Khởi tạo vùng nhớ chia sẻ, nạp cấu hình và kết thúc an toàn.
    
- **Lắng nghe & Xác thực kết nối**: Chờ kết nối mới trên cổng mặc định (port `5432`), xác thực quyền và phân nhánh (`fork`) một tiến trình con **Backend Server Process** để phục vụ riêng client đó.
    
- **Khôi phục sự cố (Crash Recovery)**: Nếu một backend worker gặp sự cố bộ nhớ, Postmaster sẽ kích hoạt dọn dẹp và khôi phục hệ thống qua WAL.
    
- **Quản lý Background Processes**: Khởi chạy và giám sát các tiến trình chạy nền.

#### Vùng Nhớ Dùng Chung (Shared Memory)

Vùng RAM được cấp phát khi khởi động instance để tất cả tiến trình con truy cập đồng thời:

- **Shared Disk Buffer (Buffer Pool)**:
    
    - _Mục tiêu_: Lưu trữ tạm thời các trang dữ liệu (`8 KB`) nhằm giảm thiểu thao tác Disk I/O.
        
    - Tăng tốc độ đọc/ghi dữ liệu lớn, tránh tình trạng quá tải ổ đĩa khi nhiều client truy cập đồng thời.
        
- **WAL Buffers**: Vùng đệm chứa các bản ghi nhật ký giao dịch trước khi được ghi xuống đĩa.
    
- **Lock Table & IPC**: Bảng khóa toàn cục dùng chung để đồng bộ và kiểm soát đồng thời giữa các backend process.

#### Tiến trình Backend (Backend Process) & Các Tiến trình Nền (Background Processes)

- **Backend Server Process**:
    
    - Nhận câu lệnh SQL, phân tích cú pháp, lập kế hoạch thực thi và lấy dữ liệu từ `Shared Buffer` hoặc đĩa.
        
    - Mỗi backend process xử lý tuần tự một truy vấn tại một thời điểm cho một client tương ứng.
        
- **Các Tiến trình Nền (Background Processes cốt lõi)**:
    
    - **WAL Writer**: Định kỳ ghi và xả dữ liệu từ `WAL Buffer` xuống đĩa cứng.
        
    - **Checkpointer**: Khi sự kiện checkpoint xảy ra, chịu trách nhiệm xả toàn bộ các trang dữ liệu bị sửa đổi (_dirty buffers_) từ RAM xuống tệp tin dữ liệu.
        
    - **Background Writer (bgwriter)**: Chạy định kỳ để xả các trang bẩn (_dirty pages_) xuống đĩa một cách nhẹ nhàng, đảm bảo luôn có sẵn các trang trống (_clean buffers_) cho backend process sử dụng.
        
    - **Autovacuum Launcher**: Khởi tạo các `autovacuum worker` để dọn dẹp bảng phình to (_bloat_), xóa dead tuples và đóng băng XID.
        
    - **Logging Collector (Logger)**: Thu thập thông báo lỗi, cảnh báo từ tất cả các tiến trình và ghi vào tệp log.
        
    - **Archiver**: Khi chế độ archiving được bật, tiến trình này sao chép các đoạn tệp WAL đã đầy sang thư mục lưu trữ/backup dự phòng.
        
    - **Stats Collector**: Thu thập số liệu thống kê hoạt động của hệ thống (phục vụ các view giám sát như `pg_stat_activity`, `pg_stat_all_tables`).

#### So Sánh với Vùng Nhớ "Shared Pool" (Oracle vs. PostgreSQL)

- **Khác biệt**: PostgreSQL **không sử dụng Shared Pool** tập trung (như của kiến trúc Oracle SGA) để cache mã thực thi SQL toàn cục cho mọi người dùng.
    
- **Cơ chế của PostgreSQL**:
    
    - Quản lý parse/plan ở **mức tiến trình (Process Level)**.
        
    - Truy vấn chạy nhiều lần trong cùng một phiên kết nối (_Prepared Statement_) chỉ cần **hard-parse một lần**, sau đó tái sử dụng Execution Plan trực tiếp trong bộ nhớ riêng của tiến trình đó.
        
    - _Lợi ích_: Giảm thiểu tranh chấp khóa bộ nhớ (_latch contention_) trên vùng nhớ dùng chung khi có tải truy vấn đồng thời cao.

### Đặc Điểm & Tính Năng Chuyên Biệt

- **OID (Object Identifier)**:
    
    - Là số nguyên không dấu 4-byte (`uint32`) dùng làm khóa chính định danh các đối tượng trong catalog hệ thống (`pg_class`, `pg_type`,...).
        
    - _Lưu ý_: Từ **PostgreSQL 12**, tùy chọn tạo bảng người dùng với OID (`WITH OIDS`) đã bị loại bỏ hoàn toàn; bảng người dùng nên sử dụng khóa chính thông thường (`SERIAL`, `BIGINT`, `UUID`).
        
- **Vacuuming & Bảo trì**: Tự động dọn dẹp dead tuples sinh ra do MVCC, tối ưu hóa kích thước đĩa và ngăn chặn nguy cơ tràn số định danh giao dịch (_XID Wraparound_).

### Đánh Giá Ưu Điểm & Hạn Chế

| **Khía cạnh** | **Điểm nổi bật**                                                                                                                                                                                                                                                                                                          |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Ưu điểm**   | • Chuẩn ACID nghiêm ngặt, độ tin cậy và khả năng chịu lỗi cao.<br><br>• Mô hình Process-per-connection giúp cách ly lỗi tuyệt đối (lỗi crash của một backend không làm sập instance).<br><br>• Hỗ trợ mở rộng kiểu dữ liệu mạnh mẽ (JSONB, GIS, Custom Extensions).<br><br>• Hệ sinh thái nguồn mở và tài liệu phong phú. |
| **Hạn chế**   | • Mô hình Process tốn RAM và chi phí Context Switch cao hơn Thread khi có hàng nghìn kết nối đồng thời $\rightarrow$ cần dùng Connection Pooler như **PgBouncer**.<br><br>• Dead tuples sinh ra bởi MVCC đòi hỏi quy trình bảo trì (Vacuum) liên tục để tránh bloat bảng và chỉ mục.                                      |

## Note

Là Heap Table

Dữ liệu được lưu ngẫu nhiên trên disk

- Khi INSERT, dữ liệu được thêm vào chỗ trống bất kì trên disk

- Khi DELETE, UPDATE, dữ liệu sẽ chưa được xóa ngay mà phải chờ được dọn dẹp (VACUUM, ...) 

Dữ liệu khi được ghi, cập nhật rất nhưng đổi lại khi tìm kiếm mà không có index thì rất chậm

- Khi tìm kiếm dữ liệu, postgreSQL phải tìm kiếm toàn bảng khi không có index

Không hỗ trợ nhiều storage engine

Hoạt động dựa trên cơ chế MVCC mạnh mẽ, cung cấp cho các bản ghi một ctid

Mỗi 1 bảng (không phân vùng) bao gồm 1 file data và các file index riêng

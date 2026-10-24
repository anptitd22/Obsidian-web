---
title: Backup & Recovery
tags:
  - MySQL
---
## Phân loại
### Phân loại theo Trạng thái hoạt động (State)

- **Cold Backup (Offline):** Tắt hẳn MySQL server rồi mới backup.
    
    - _Đặc điểm:_ An toàn, đơn giản nhất nhưng phải chịu **downtime**.
        
- **Warm Backup:** MySQL vẫn chạy nhưng đặt chế độ **chỉ đọc (Read-only)**.
    
    - _Đặc điểm:_ Vẫn đọc được (`SELECT`), nhưng bị chặn toàn bộ thao tác ghi (`INSERT`, `UPDATE`, `DELETE`).
        
- **Hot Backup (Online):** MySQL chạy bình thường, phục vụ **cả đọc lẫn ghi**.
    
    - _Đặc điểm:_ **Zero downtime**, phù hợp cho hệ thống Production 24/7.

### Phân loại theo Bản chất dữ liệu (Format)

- **Logical Backup (Sao lưu logic):**
    
    - _Cách làm:_ Đọc dữ liệu rồi xuất thành các câu lệnh SQL (`CREATE TABLE`, `INSERT INTO...`) hoặc file text (`CSV`).
        
    - _Công cụ:_ `mysqldump`, `mysqlpump`, `mydumper`.
        
    - _Đặc điểm:_ File đọc/sửa được, dễ migrate giữa các phiên bản; nhược điểm là tốc độ backup và restore **chậm khi dữ liệu lớn**.
        
- **Physical Backup (Sao lưu vật lý):**
    
    - _Cách làm:_ Sao chép trực tiếp các file nhị phân, cấu trúc thư mục dữ liệu (`.ibd`, tablespace, file log).
        
    - _Công cụ:_ Copy thư mục `datadir` (khi cold backup), Percona XtraBackup, MySQL Enterprise Backup.
        
    - _Đặc điểm:_ Tốc độ backup và restore **cực nhanh**, phù hợp cho database dung lượng lớn (hàng trăm GB đến TB).

### Phân loại theo Khối lượng dữ liệu (Scope)

- **Full Backup:** Sao lưu toàn bộ dữ liệu hiện có trong database tại thời điểm đó.
    
- **Differential Backup:** Chỉ sao lưu phần dữ liệu **thay đổi so với bản Full Backup gần nhất**.
    
- **Incremental Backup:** Chỉ sao lưu phần dữ liệu **thay đổi so với lần Backup gần nhất** (bất kể lần trước là Full hay Incremental).
    
    - Trong MySQL, Incremental Backup thường kết hợp cơ chế đọc các file **Binary Log (Binlog)** để tái tạo lại các transaction mới nhất (Point-in-Time Recovery).

---

## Phương pháp

### mysqldump (Logical Backup)

- **Bản chất:** Trích xuất cấu trúc và dữ liệu thành các câu lệnh SQL.
    
- **Hỗ trợ:** Mọi loại bảng/storage engine.
    
- **Tùy chọn Hot Backup cho InnoDB:** Sử dụng cờ `--single-transaction` để backup trực tuyến mà **không cần lock bảng** (dựa vào cơ chế snapshot/MVCC của InnoDB).

## MySQL Enterprise Backup (Physical Backup)

- **Bản chất:** Sao chép tệp vật lý trực tiếp từ đĩa (hỗ trợ nén và sao lưu gia tăng - incremental).
    
- **Ưu điểm lớn nhất:** Tốc độ **Restore nhanh hơn rất nhiều** so với các công cụ logic như `mysqldump`.
    
- **Cơ chế:**
    
    - Với bảng **InnoDB**: Thực hiện cơ chế **Hot Backup** (không khóa bảng, vẫn đọc/ghi bình thường).
        
    - Với các storage engine khác: Dùng cơ chế **Warm Backup** (khóa bảng chỉ đọc).

### Copy file dữ liệu trực tiếp (Copying Table Files)

- **Phù hợp:** Chủ yếu cho bảng **MyISAM** (`.MYD`, `.MYI`, `.sdi`).
    
- **Quy trình an toàn:** Tắt server hoặc khóa bảng và flush dữ liệu xuống đĩa:

```sql
    FLUSH TABLES tbl_list WITH READ LOCK;
    -- Tiến hành copy file
    UNLOCK TABLES;
```

- **Cảnh báo với InnoDB:** **Không áp dụng cách copy file thủ công** cho InnoDB khi server đang chạy, vì InnoDB lưu rất nhiều dữ liệu bẩn chưa flush trên Buffer Pool (RAM).

### Delimited-Text File Backup (CSV / Text thuần)

- **Cách 1 - Lệnh SQL:** `SELECT * INTO OUTFILE 'file_name' FROM tbl_name;`
    
    - File được sinh ra trực tiếp trên **máy chủ MySQL** (server host).
        
    - Yêu cầu file xuất ra chưa từng tồn tại (đảm bảo bảo mật).
        
    - _Nhược điểm:_ Chỉ lưu dữ liệu hàng, **không lưu cấu trúc bảng (schema)**.
        
- **Cách 2 - mysqldump:** Dùng tùy chọn `mysqldump --tab` để xuất đồng thời cả file text dữ liệu và file `.sql` chứa lệnh `CREATE TABLE`.
    
- **Nạp lại dữ liệu:** Sử dụng `LOAD DATA INFILE` hoặc công cụ dòng lệnh `mysqlimport`.

### Incremental Backup bằng Binary Log (Binlog)

- **Bản chất:** Kết hợp một bản Full Backup với các file Binary Log sinh ra sau thời điểm backup đó để phục hồi dữ liệu tới một mốc thời gian cụ thể (Point-in-Time Recovery).

	- Backup theo vị trí trên binlog

	- Backup theo thời gian trên binlog

- **Lưu ý**:

	- Có thể backup về trạng thái trước đó sau khi đã thực thi câu sql

	- Nên chọn backup theo vị trí 

	- Nên kết hợp việc backup full data + Incremental theo vị trí để có đầy đủ dữ liệu
    
- **Quy trình:**
    
    1. Xoay vòng log bằng lệnh: `FLUSH LOGS;` (hoặc cờ `mysqldump --flush-logs`).
        
    2. Sao lưu toàn bộ các file binlog từ sau mốc backup gần nhất đến file vừa được xoay vòng.

### Backup thông qua Replica (Slave)

- **Lợi ích:** 

	- Tránh gây sụt giảm hiệu năng (I/O đĩa, CPU) trên máy chủ chính (Source/Master).

	- Slave có thể lên thay Master khi gặp lỗi

	- Zero downtime, cung cấp HA và khuyên dùng trên PROD
    
- **Các thành phần bắt buộc phải backup kèm trên Replica:**
    
    - **Connection & Applier Metadata Repositories:** Thông tin vị trí replication để sau khi khôi phục có thể tiếp tục đồng bộ từ Master.
        
    - Các file tạm `SQL_LOAD-*` nằm trong thư mục `replica_load_tmpdir` (nếu có các lệnh `LOAD DATA` đang chạy dở dang).

### File System Snapshot (LVM / Veritas VXFS / ZFS)

- **Bản chất:** Tận dụng tính năng chụp nhanh (snapshot) ở tầng file system/OS để giảm thiểu thời gian khóa bảng xuống mức mili-giây.
    
- **Quy trình 5 bước:**
    
    1. Khóa ghi: `FLUSH TABLES WITH READ LOCK;`
        
    2. Tạo và mount snapshot ở tầng hệ điều hành.
        
    3. Mở khóa ngay lập tức: `UNLOCK TABLES;` (ứng dụng tiếp tục đọc/ghi bình thường).
        
    4. Copy dữ liệu từ phân vùng snapshot ra nơi lưu trữ an toàn.
        
    5. Unmount snapshot.
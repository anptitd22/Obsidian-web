Năm 2016, đội ngũ kỹ thuật của Uber đăng tải bài viết nổi tiếng lý giải nguyên nhân chuyển từ PostgreSQL sang MySQL (với storage engine InnoDB) để xây dựng hệ thống sharding riêng mang tên **Schemaless**.

### 1. "Bài toán của Uber" là gì?

Workload của Uber tại thời điểm đó có những đặc thù rất khắc nghiệt:

- **Tần suất ghi (Write-heavy) và cập nhật cực cao:** Vị trí xe, tọa độ tài xế, trạng thái cuốc xe (matching, di chuyển, thanh toán) thay đổi liên tục theo từng giây trên quy mô toàn cầu.
    
- **Yêu cầu không downtime:** Dịch vụ hoạt động 24/7/365, không thể tạm dừng để nâng cấp phiên bản hay bảo trì bảng dữ liệu.
    
- **Nhu cầu mở rộng ngang (Horizontal Sharding):** Lượng dữ liệu vượt quá giới hạn scale-up của một máy chủ vật lý, buộc phải phân mảnh dữ liệu (sharding).
    

### 2. Tại sao PostgreSQL gặp vấn đề ở quy mô của Uber?

Quyết định rời bỏ Postgres không phải vì Postgres tệ, mà vì các cơ chế nội tại thời điểm đó (Postgres 9.x) xung đột với mô hình dữ liệu ghi/cập nhật liên tục của Uber:

#### Hiện tượng khuếch đại ghi (Write Amplification) và Index Bloat

- **Cơ chế Heap Tuple của PostgreSQL:** Trong Postgres, mỗi bản ghi (row tuple) được đánh dấu bằng một con trỏ vật lý gọi là `ctid` (trỏ trực tiếp vào `Block/Offset` trên ổ đĩa).
    
- Khi thực hiện lệnh `UPDATE`, do mô hình MVCC của Postgres là bất biến (immutable), nó không sửa tại chỗ mà ghi một tuple mới ở vị trí đĩa mới (mang `ctid` mới).
    
- Hậu quả: **Tất cả các secondary indexes đều phải cập nhật** để trỏ sang `ctid` mới (trừ khi thỏa mãn điều kiện HOT – Heap-Only Tuples rất khắt khe). Nếu một bảng có 5–6 index, một thao tác cập nhật nhỏ có thể biến thành 6–7 lần ghi đĩa, làm phình to (bloat) index và nghẽn I/O.
    

#### Cơ chế Replication (Physical WAL Replication)

- Thời điểm đó, Postgres chủ yếu dùng cơ chế replication vật lý dựa trên Write-Ahead Log (WAL).
    
- Mỗi thay đổi nhỏ ở tầng byte trên đĩa (kể cả việc cập nhật lại secondary index do đổi `ctid`) đều sinh ra log WAL và phải truyền qua mạng tới toàn bộ các node replica. Điều này tạo ra áp lực băng thông mạng khổng lồ giữa các datacenter của Uber.
    

#### Xung đột MVCC trên Replica (Replica MVCC Conflict)

- Do replica nhận các thay đổi vật lý trực tiếp qua WAL, nếu Master chạy tiến trình `VACUUM` dọn dẹp các tuple cũ, WAL log sẽ gửi lệnh xóa các tuple đó sang Replica.
    
- Nếu trên Replica đang có truy vấn phân tích (read query) chạy đọc qua tuple đó, truy vấn sẽ bị xung đột và Postgres buộc phải hủy (kill) truy vấn hoặc hoãn áp dụng WAL, dẫn đến **Replication Lag** nghiêm trọng.
    

#### Nâng cấp phiên bản phức tạp và đòi hỏi downtime

- Việc nâng cấp giữa các phiên bản lớn (major version) của Postgres giai đoạn đó rất cồng kềnh: chạy `pg_upgrade`, dừng master (downtime), đồng bộ lại snapshot hàng terabyte sang toàn bộ các replicas.
    

### 3. MySQL (InnoDB) giải quyết bài toán đó như thế nào?

Uber chuyển sang sử dụng MySQL/InnoDB làm storage engine bên dưới tầng sharding **Schemaless** nhờ các ưu điểm kiến trúc:

|**Yếu tố kiến trúc**|**PostgreSQL (thời điểm 9.x)**|**MySQL (InnoDB)**|
|---|---|---|
|**Cấu trúc bảng & Index**|Heap file; Secondary index trỏ trực tiếp vào vị trí vật lý (`ctid`) trên đĩa.|Clustered Index (dữ liệu nằm trong B+Tree của Primary Key); Secondary index trỏ vào Primary Key.|
|**Ảnh hưởng của `UPDATE`**|Đổi vị trí đĩa $\rightarrow$ Phải cập nhật toàn bộ secondary indexes $\rightarrow$ Write amplification cao.|Cập nhật dữ liệu tại chỗ hoặc qua Change Buffer; chỉ index của cột bị sửa mới cần cập nhật.|
|**Replication**|Physical WAL (sao chép trực tiếp từng thay đổi byte trên disk).|Logical Row-based Binlog (chỉ ghi nhận sự thay đổi giá trị logic của hàng) $\rightarrow$ Tiết kiệm băng thông đáng kể.|
|**MVCC**|Lưu các version cũ trực tiếp trong table heap data $\rightarrow$ Cần tiến trình `VACUUM`.|Tách biệt version mới trong Clustered Index và version cũ trong **Undo Log** $\rightarrow$ Tránh phình bảng chính.|

### Bối cảnh hiện đại

Từ PostgreSQL 10 đến các bản phát hành hiện nay, Postgres đã bổ sung **Logical Replication**, cải tiến tối ưu hóa HOT update, nén và khử trùng lặp index B-tree, cùng cơ chế vacuum song song nhằm giảm bớt đáng kể các nhược điểm trên. Quyết định của Uber là kết quả của việc tối ưu hóa cho một bài toán hạ tầng rất hẹp: hệ thống phân tán append/write-heavy với tầng sharding tự xây dựng (Schemaless) ở quy mô hàng trăm triệu chuyến đi.
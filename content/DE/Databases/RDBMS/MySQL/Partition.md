Tính năng phân vùng được hỗ trợ bởi các công cụ lưu trữ InnoDB và NDB.

## Bản chất & Nguyên lý cốt lõi

- **Khái niệm:** Phân vùng bảng (Partitioning) chia nhỏ một bảng logic lớn thành nhiều phần vật lý độc lập (gọi là các _partitions_), nhưng tầng ứng dụng (SQL) vẫn nhìn nhận và thao tác như **một bảng duy nhất**.
    
- **Phân loại ở mức lưu trữ:** MySQL chỉ hỗ trợ **Horizontal Partitioning** (chia theo các hàng/records), không hỗ trợ _Vertical Partitioning_ (chia cột) ở mức cấu hình phân vùng engine.
    
- **Lưu trữ trên đĩa (InnoDB):** Mỗi partition được lưu thành một tệp tablespace riêng biệt (`.ibd`), giúp giảm kích thước cây B+Tree của từng phần, giảm phân mảnh và tăng tốc độ I/O.

## Lợi ích

- **Cắt tỉa phân vùng (Partition Pruning):**
    
    - Trình tối ưu hóa truy vấn (_Query Optimizer_) tự động phát hiện điều kiện lọc (`WHERE`) và **chỉ quét các partition liên quan**, bỏ qua hoàn toàn các partition còn lại.
        
    - Giảm số lượng disk pages cần nạp vào Buffer Pool.
        
- **Bảo trì & Vòng đời dữ liệu (Data Archiving / Purging):**
    
    - Xóa dữ liệu cũ trong bảng hàng trăm triệu dòng bằng `DELETE` sinh ra khối lượng Redo/Undo log khổng lồ và gây lock bảng.
        
    - Với partitioning, việc dọn dữ liệu chỉ mất chưa đầy 1 giây:

```sql
ALTER TABLE logs DROP PARTITION p2023; -- Xóa toàn bộ tệp vật lý, zero impact
```
    
- **Quản trị độc lập:** Có thể bảo trì, chống phân mảnh (`OPTIMIZE PARTITION`), kiểm tra lỗi (`CHECK PARTITION`) trên từng phần mà không ảnh hưởng toàn bảng.

## Các loại Partitioning trong MySQL

### RANGE Partitioning (Phổ biến nhất)

Phân vùng theo các dải giá trị liên tục, không chồng chéo. Phù hợp nhất cho dữ liệu theo thời gian (logs, orders, transactions).

```sql
CREATE TABLE orders (
    order_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10,2),
    PRIMARY KEY (order_id, order_date)
)
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### LIST Partitioning

Phân vùng theo danh sách các giá trị cụ thể, rời rạc (enum, country code, status).

```sql
CREATE TABLE users (
    id INT NOT NULL,
    country_code VARCHAR(2) NOT NULL,
    name VARCHAR(50),
    PRIMARY KEY (id, country_code)
)
PARTITION BY LIST COLUMNS(country_code) (
    PARTITION p_asia VALUES IN ('VN', 'TH', 'SG', 'MY'),
    PARTITION p_europe VALUES IN ('UK', 'DE', 'FR'),
    PARTITION p_america VALUES IN ('US', 'CA')
);
```

### HASH Partitioning

Dùng hàm băm để phân bổ dữ liệu đồng đều vào một số lượng partition xác định trước, tránh điểm nóng ghi (_write hotspots_).

```sql
CREATE TABLE user_sessions (
    session_id INT NOT NULL,
    user_id INT NOT NULL,
    created_at DATETIME,
    PRIMARY KEY (session_id, user_id)
)
PARTITION BY HASH(user_id)
PARTITIONS 4;
```

### KEY Partitioning

Tương tự như `HASH`, nhưng hàm băm được tính trực tiếp bởi MySQL nội bộ (hỗ trợ kiểu dữ liệu không phải integer) và mặc định sử dụng Primary Key.

```sql
CREATE TABLE cache_data (
    cache_key VARCHAR(64) NOT NULL PRIMARY KEY,
    cache_val TEXT
)
PARTITION BY KEY()
PARTITIONS 8;
```

### Subpartitioning (Composite Partitioning)

Chia nhỏ tiếp một partition thành các subpartition (thường kết hợp `RANGE - HASH` hoặc `RANGE - KEY`).

## Lưu ý

- **Ràng buộc Primary Key & Unique Key:**
    
    - Mọi cột nằm trong Primary Key hoặc bất kỳ Unique Index nào của bảng **bắt buộc phải có mặt trong Partitioning Expression / Partitioning Key**.
        
    - _Hệ quả:_ Nếu bạn muốn partition bảng theo `created_at`, nhưng khóa chính chỉ có `id`, MySQL sẽ **từ chối tạo bảng** trừ khi đổi khóa chính thành `PRIMARY KEY (id, created_at)`.
        
- **Không có Global Indexes:**
    
    - MySQL chỉ hỗ trợ **Local Partitioned Index** (mỗi index chỉ thuộc về 1 partition). MySQL không hỗ trợ _Global Index_.
        
    - Do đó, mọi truy vấn tìm kiếm không truyền kèm Partitioning Key sẽ buộc phải quét toàn bộ các partitions (**Scatter-Gather Query**), gây suy giảm hiệu năng nghiêm trọng.
        
- **Giới hạn số lượng:** Tối đa 8192 partitions trên một bảng (bao gồm cả subpartitions).
    
- **Foreign Key Restriction:** Các bảng dùng partitioning **không hỗ trợ ràng buộc khóa ngoại (Foreign Keys)** trong InnoDB.
    
- **Fulltext & Spatial Index:** Không được hỗ trợ trên bảng có partition (ở các phiên bản MySQL cũ, cần kiểm tra kỹ nếu dùng MySQL 8.0).

## Kiểm tra & Giám sát Partition Pruning

Để kiểm tra xem một truy vấn có tận dụng được cơ chế cắt tỉa hay không, dùng lệnh `EXPLAIN`:

```sql
EXPLAIN SELECT * FROM orders WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';
```

**Cột `partitions`:** Nếu cột này chỉ hiển thị `p2024` thay vì toàn bộ `p2023,p2024,p2025,p_future`, nghĩa là truy vấn đã được **Partition Pruning** thành công.
### 1. Vấn đề gốc của MVCC đối với lệnh `UPDATE`

- **Cơ chế:** PostgreSQL xử lý `UPDATE` bằng cách thêm một phiên bản hàng mới (tuple mới/ctid mới) thay vì ghi đè trực tiếp.
    
- **Hệ quả tiêu cực:**
    
    - Mỗi tuple mới thường kéo theo việc chèn thêm các bản ghi index mới vào **tất cả** các index của bảng.
        
    - Việc dọn dẹp các tuple cũ và index pointer tương ứng tốn nhiều tài nguyên I/O (`VACUUM`).
        
    - Gây ra hiện tượng phình to dữ liệu (table/index bloat) và khuếch đại ghi (write amplification).
        

### 2. Giải pháp: Tối ưu hóa Heap-Only Tuples (HOT)

HOT là cơ chế cho phép tạo tuple mới ngay trong trang (page) chứa tuple cũ mà **không cần tạo thêm index entry mới**.

#### Điều kiện để kích hoạt HOT:

1. **Không sửa cột có index:** Lệnh `UPDATE` không làm thay đổi bất kỳ cột nào được đánh index (ngoại trừ _summarizing indexes_ như **BRIN**).
    
2. **Trang còn đủ chỗ trống:** Data page hiện tại chứa phiên bản cũ phải có đủ dung lượng khả dụng để chứa phiên bản mới.
    

#### Hai lợi ích cốt lõi khi thỏa mãn điều kiện:

- **Không sinh thêm index entry:** Các secondary index giữ nguyên, tiếp tục trỏ vào vị trí ban đầu (chỉ BRIN index có thể cần cập nhật tóm tắt).
    
- **Tự động dọn dẹp (Pruning on-the-fly):**
    
    - Không cần chờ tiến trình `VACUUM` chạy định kỳ. Các truy vấn thông thường (kể cả lệnh `SELECT`) có thể trực tiếp dọn dẹp các phiên bản trung gian không còn transaction nào nhìn thấy.
        
    - Item identifier cũ được chuyển thành một con trỏ chuyển hướng (**redirect pointer**) liên kết trực tiếp trong page chain.
        

### 3. Thực hành cấu hình & Giám sát (Tuning & Monitoring)

#### Tăng xác suất xảy ra HOT bằng `fillfactor`:

- Mặc định `fillfactor = 100` (trang được lấp đầy khi `INSERT`), khiến page không còn khoảng trống cho `UPDATE`.
    
- **Khuyến nghị:** Giảm `fillfactor` (ví dụ: `80` hoặc `70`) trên các bảng có tần suất `UPDATE` cao để chừa khoảng trống (headroom) cho tuple mới nằm cùng page:
    

SQL

```
ALTER TABLE table_name SET (fillfactor = 80);
-- Cần VACUUM FULL hoặc REINDEX/pg_repack để áp dụng cấu trúc trang mới
```

#### Giám sát hiệu quả HOT:

Theo dõi tỷ lệ HOT update qua system catalog `pg_stat_all_tables`:

SQL

```
SELECT 
    relname,
    n_tup_upd,
    n_tup_hot_upd,
    ROUND(n_tup_hot_upd::numeric / NULLIF(n_tup_upd, 0) * 100, 2) AS hot_ratio_pct
FROM pg_stat_all_tables
WHERE n_tup_upd > 0
ORDER BY n_tup_upd DESC;
```

- **Mục tiêu:** `hot_ratio_pct` càng gần 100% càng tốt cho các bảng ghi/cập nhật liên tục.
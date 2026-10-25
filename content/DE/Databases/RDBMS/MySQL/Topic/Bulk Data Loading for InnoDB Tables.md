---
title: Bulk Data Loading for InnoDB
tags:
  - MySQL
  - Topic
  - RDBMS
---
## 1. Tối ưu Session & Transaction (Giảm Disk I/O)

- **Tắt Autocommit:** Gom toàn bộ lệnh nạp vào một transaction lớn thay vì để MySQL ghi đĩa (flush log) sau mỗi dòng chèn.

```sql
SET autocommit = 0;
-- Các câu lệnh nạp dữ liệu (INSERT / LOAD DATA) ...
COMMIT;
```

- **Tạm tắt kiểm tra khóa ngoại (Foreign Key):** Giảm tải việc tra cứu chéo sang các bảng cha.

```sql
SET foreign_key_checks = 0;
-- ... Nạp dữ liệu ...
SET foreign_key_checks = 1;
```
- **Tạm tắt kiểm tra trùng lặp khóa phụ (Unique Checks):** Cho phép InnoDB tận dụng **Change Buffer** để ghi gộp các secondary index theo batch thay vì ghi ngẫu nhiên từng dòng.

```sql
SET unique_checks = 0;
-- ... Nạp dữ liệu (đảm bảo file nguồn không có duplicate) ...
SET unique_checks = 1;
```

## 2. Tối ưu câu lệnh & Dữ liệu nguồn

- **Multi-row INSERT:** Dùng cú pháp `INSERT INTO tbl VALUES (...), (...), (...)` để giảm độ trễ mạng và overhead phân tích cú pháp giữa Client và Server, đảm bảo việc insert được đầy đủ không bị thiếu.
    
- **Sắp xếp sẵn theo PRIMARY KEY:**
    
    - Do InnoDB lưu trữ dữ liệu theo cấu trúc **Clustered Index**, việc nạp dữ liệu đã được sort theo thứ tự khóa chính giúp ghi tuần tự vào các data page cuối, hạn chế tối đa hiện tượng phân mảnh và tách trang (**Page Split**).
        
- **Cấu hình AUTO_INCREMENT:** Đặt `innodb_autoinc_lock_mode = 2` (Interleaved) để loại bỏ khóa bảng `AUTO-INC`, hỗ trợ nhiều luồng ghi dữ liệu đồng thời.

## 3. Tối ưu cho chỉ mục FULLTEXT (FTS)

Nếu nạp bảng có Full-Text Search, làm theo 3 bước chuẩn:

1. **Tạo bảng kèm cột định danh FTS thủ công:** Định nghĩa cột `FTS_DOC_ID BIGINT UNSIGNED NOT NULL` với unique index `FTS_DOC_ID_INDEX` (giúp tránh việc InnoDB phải tự tạo cột ẩn và rebuild lại bảng sau này).
    
2. **Nạp toàn bộ dữ liệu vào trước.**
    
3. **Tạo FULLTEXT Index sau cùng:** Xây dựng index một lần trên toàn bộ tập dữ liệu nhanh hơn nhiều so với việc vừa chèn từng dòng vừa cập nhật cây inverted index.

### 4. Tối ưu tầng Instance & Công cụ nạp (Advanced)

- **Tắt tạm thời Redo Log (Chỉ dùng cho DB mới dựng / Initial Load):**

```sql
ALTER INSTANCE DISABLE INNODB REDO_LOG;
-- ... Nạp dữ liệu ...
ALTER INSTANCE ENABLE INNODB REDO_LOG;
```

- Cảnh báo nguy hiểm: **Tuyệt đối không dùng trên Production**. Nếu server crash/mất điện trong lúc đang tắt Redo Log, toàn bộ instance có nguy cơ hỏng không thể phục hồi.

- **Sử dụng công cụ song song (MySQL Shell):**
    
    - Dùng `util.importTable()` hoặc `util.loadDump()` trong **MySQL Shell** để chia nhỏ file dữ liệu và nạp đa luồng (multi-threaded parallel load), cho tốc độ vượt trội so với import qua client thông thường.

Liên quan:
- [[Buffer Pool]]
- [[Glossary]]
- [[Log]]
- [[DE/Databases/RDBMS/MySQL/Indexes]]

---
title: Indexes
tags:
  - MySQL
---
## Clustered Index & Secondary Index

**Bản chất Clustered Index (Chỉ mục phân cụm):**

- Là nơi lưu trữ trực tiếp toàn bộ dữ liệu thực tế của hàng (Leaf node = Dữ liệu hàng).
    
- Mỗi bảng **bắt buộc và chỉ có 1** Clustered Index.

**Quy tắc InnoDB chọn Clustered Index (theo thứ tự ưu tiên):**

1. `PRIMARY KEY` (Khóa chính do bạn tạo).
    
2. Index `UNIQUE` đầu tiên mà tất cả cột đều `NOT NULL`.
    
3. Nếu không có cả hai: InnoDB tự tạo cột ẩn `GEN_CLUST_INDEX` (Row ID 6-byte, tự tăng dần).

**Vì sao Clustered Index nhanh?**

- Tìm kiếm trỏ thẳng đến data page chứa hàng đó thay vì phải chạy tìm kiếm toàn bảng như Non-index (PostgreSQL)

- Các dữ liệu đã được sắp xếp nên khi tìm kiếm theo Sequential I/O rất nhanh, 

**Mối quan hệ với Secondary Index (Chỉ mục phụ):**

- Leaf node của Secondary Index không chứa trực tiếp dữ liệu hàng, mà **chỉ chứa giá trị của Primary Key**.
    
- Quy trình tìm kiếm qua Secondary Index: Tìm trong Secondary Index lấy Primary Key $\rightarrow$ Tra cứu lại vào Clustered Index để lấy dữ liệu đầy đủ (_quá trình Lookup/Bookmark Lookup_).

**Lưu ý thiết kế (Best Practices):**

- **Luôn tạo Primary Key** (ưu tiên dạng số nguyên tự tăng `AUTO_INCREMENT`).
    
- **Giữ Primary Key càng ngắn càng tốt:** Vì Primary Key sẽ bị nhân bản vào mọi Secondary Index; Primary Key dài sẽ làm phình to dung lượng đĩa và tốn RAM bộ nhớ đệm.

- Việc thay đổi dữ liệu (insert, update) sẽ tác động lên cả Primary Key lẫn Secondary Key

---
## B-tree

B-tree có thể được sử dụng để so sánh cột trong các biểu thức sử dụng các toán tử =, >, >=, <, <= hoặc BETWEEN. Index cũng có thể được sử dụng cho các phép so sánh LIKE nếu đối số của LIKE là một chuỗi hằng số không bắt đầu bằng ký tự đại diện (%). Ví dụ, các câu lệnh SELECT sau sử dụng chỉ mục:

```sql
SELECT * FROM tbl_name WHERE key_col LIKE 'Patrick%';
SELECT * FROM tbl_name WHERE key_col LIKE 'Pat%_ck%';
```

- Trong câu lệnh đầu tiên, chỉ những hàng có 'Patrick' <= key_col < 'Patricl' được xem xét. Trong câu lệnh thứ hai, chỉ những hàng có 'Pat' <= key_col < 'Pau' được xem xét.

Các câu lệnh SELECT sau đây không sử dụng chỉ mục:

```sql
SELECT * FROM tbl_name WHERE key_col LIKE '%Patrick%';
SELECT * FROM tbl_name WHERE key_col LIKE other_col;
```

>[!note]
>Bản chất của B-tree là sort các cột được đánh index rồi tìm kiếm

>[!warning]
>Việc đánh index nhiều cột (A, B, C) thì khi tìm kiếm phải so sánh có cột A ví dụ: A hoặc (A, B) hoặc (A, C) hoặc (A, B, C) nếu không thì sẽ không thể sử dụng index

Các bước tìm kiếm: 
- Tìm kiếm giá trị 120

![[Pasted image 20261024013633.png]]

- Step1:

![[Pasted image 20261024013700.png]]

- Step 2:

![[Pasted image 20261024013711.png]]

- Step 3:

![[Pasted image 20261024013723.png]]

- Step 4:

![[Pasted image 20261024013802.png]]

>[!Note]
>Trên chỉ là minh họa thuật toán còn thực tế thì thường nó sẽ đi kèm với vị trí của các page

## Hash Index

- Chúng chỉ được sử dụng cho các phép so sánh bằng nhau sử dụng toán tử = hoặc <=> (nhưng rất nhanh). Chúng không được sử dụng cho các toán tử so sánh như < dùng để tìm một phạm vi giá trị. Các hệ thống dựa vào kiểu tra cứu giá trị đơn được gọi là "key-value"; để sử dụng MySQL cho các ứng dụng như vậy, hãy sử dụng hash index bất cứ khi nào có thể.

- Dữ liệu index được tổ chức theo dạng Key - Value được liên kết với nhau.

- Không thể tối ưu hóa toán tử ORDER BY bằng việc sử dụng HASH index bởi vì nó không thể tìm kiếm được phần từ tiếp theo trong Order.
 
 - MySQL không thể xác định gần đúng số lượng hàng nằm giữa hai giá trị (thông tin này được trình tối ưu hóa phạm vi sử dụng để quyết định sử dụng chỉ mục nào). Điều này có thể ảnh hưởng đến một số truy vấn nếu bạn chuyển đổi bảng MyISAM hoặc InnoDB thành bảng MEMORY được lập hash index.

- Chỉ có thể sử dụng toàn bộ khóa để tìm kiếm một hàng. (Với chỉ B-tree, bất kỳ tiền tố nào nằm bên trái nhất của khóa đều có thể được sử dụng để tìm hàng.)

- Độ phức tạp: O(1)

>[!note]
>Việc chọn index theo kiểu BTREE hay HASH ngoài yếu tố về mục đích sử dụng index thì nó còn phụ thuộc vào mức độ hỗ trợ của Storage Engine. Ví dụ MyISAM, InnoDB hay Archive chỉ hỗ trợ Btree, trong khi Memory lại hỗ trợ cho cả 2.

## Full-Text Index

- **Storage Engine:** InnoDB (từ MySQL 5.6+) và MyISAM.
    
- **Cơ chế:** Sử dụng kỹ thuật chỉ mục đảo ngược (Inverted Index) tương tự như GIN của Postgres hoặc Elasticsearch, chia tách chuỗi văn bản thành các từ (tokens/words) và loại bỏ stop-words.
    
- **Cú pháp truy vấn:** Sử dụng `MATCH(...) AGAINST(...)`.
    
- **Trường hợp sử dụng:** Tìm kiếm từ khóa, cụm từ trong các trường văn bản dài (`CHAR`, `VARCHAR`, `TEXT`), hỗ trợ chế độ Natural Language Search và Boolean Mode (`+word`, `-word`).

```sql
CREATE FULLTEXT INDEX idx_content ON articles(title, body);
-- Truy vấn:
SELECT * FROM articles WHERE MATCH(title, body) AGAINST('+database -nosql' IN BOOLEAN MODE);
```

## Spatial Index (R-Tree)

- **Storage Engine:** InnoDB (từ MySQL 5.7+) và MyISAM.
    
- **Cơ chế:** Dựa trên cấu trúc cây không gian **R-Tree** (tương tự như GiST trên PostGIS của Postgres).
    
- **Kiểu dữ liệu:** Các kiểu dữ liệu hình học OpenGIS như `POINT`, `LINESTRING`, `POLYGON`, `GEOMETRY`. Cột phải được định nghĩa `NOT NULL`.
    
- **Trường hợp sử dụng:** Tính toán khoảng cách tọa độ GPS, kiểm tra điểm nằm trong đa giác (`ST_Contains`, `ST_Within`).

```sql
CREATE TABLE locations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    geo POINT NOT NULL,
    SPATIAL INDEX(geo)
) ENGINE=InnoDB;
```

---

### Khi mà select 2 cột được đánh 2 index riêng biệt thì mysql sẽ chọn index nào ?

MySQL thường sử dụng index có số lượng bản ghi nhỏ nhất (index có tính chọn lọc cao nhất).
Ví dụ:
- Có index(age) và index(height)

```sql
  Select * from table where age > 50 and height > 160 
```

- Nếu index(age) > 50 có 500k bản ghi và index(height) > 160 có 300k bản ghi
- Sẽ chọn index(height)

Thông thường khi một hệ thống lớn thường dễ đến việc sử dụng sai index do cơ chế ước lượng chi phí giữa I/O sequential và I/O random (index) -> Mình phải tự chọn index bằng câu lệnh `FORCE INDEX()`

```sql
Select * from table force index(age) where age > 50 and height > 160 
```



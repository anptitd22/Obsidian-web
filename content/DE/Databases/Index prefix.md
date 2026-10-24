Tạo một index chỉ sử dụng N chữ cái đầu của cột có thể làm tệp index nhỏ hơn, tiết kiệm storage 

```sql
CREATE TABLE test (blob_col BLOB, INDEX(blob_col(10)));
```

Nhược điểm:
- Đánh đổi về performance
- Không tự động sort toàn bộ dữ liệu 
- Không đảm bảo tính unique
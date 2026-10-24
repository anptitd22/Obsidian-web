### 1. Master-Slave Replication

Trong mô hình sao chép này, một database đóng vai trò là primary server trong khi các database khác duy trì các bản sao dữ liệu của nó.

- The master database xử lý tất cả các thao tác ghi như thêm, cập nhật và xóa.

- Slave databases sao chép dữ liệu từ master chính và thường xử lý các thao tác đọc.

### 2. Master-Master Replication / Multi-Master Replication

Trong cấu hình này, nhiều databases đóng vai trò là masters và có thể chấp nhận cả thao tác đọc và ghi.

- Các thay đổi được thực hiện trong một master database sẽ được sao chép sang master databases khác.

- Cải thiện availability và cho phép thực hiện các thao tác ghi phân tán trên nhiều máy chủ.
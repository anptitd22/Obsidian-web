Đây là cơ chế được sử dụng khi hệ thống kiểm soát đồng thời đa phiên bản (MVCC) mặc định không đáp ứng được yêu cầu nghiệp vụ của ứng dụng.

## 1. Phân tích các loại khóa (Lock Concepts)

**Khóa cấp bảng (Table-Level Locks)**

- **Khái niệm:** Dù tên gọi có chứa từ "row" (như `ROW EXCLUSIVE`), tất cả các khóa ở nhóm này đều khóa trên toàn bộ bảng. Có nhiều mức độ khóa từ nhẹ nhất (`ACCESS SHARE` - cho các lệnh đọc thông thường) đến nặng nhất (`ACCESS EXCLUSIVE` - dùng khi xóa hoặc cắt xén bảng như `DROP TABLE`, `TRUNCATE`).
- **Đặc điểm:** Các khóa không xung đột có thể được nhiều giao dịch giữ cùng lúc, nhưng các khóa xung đột sẽ khiến giao dịch đến sau phải chờ. Đáng chú ý, chỉ duy nhất khóa `ACCESS EXCLUSIVE` mới có khả năng chặn các câu lệnh `SELECT` (không có tùy chọn `FOR UPDATE/SHARE`). Các khóa này được giải phóng khi kết thúc giao dịch hoặc khi lệnh quay lui (rollback) về một điểm lưu (savepoint) được kích hoạt.

**Khóa cấp hàng (Row-Level Locks)**

- **Khái niệm:** Các lệnh truy vấn có thêm `FOR UPDATE`, `FOR SHARE`... sẽ yêu cầu khóa cấp hàng. Khác với khóa bảng, khóa hàng không cản trở việc đọc dữ liệu (truy vấn `SELECT` thông thường).
- **Đặc điểm:** Nó chỉ chặn các giao dịch khác có ý định ghi (thay đổi/xóa) hoặc muốn khóa (lock) cùng một hàng dữ liệu đó.

**Khóa cấp trang (Page-Level Locks)**

- **Khái niệm:** Hệ thống sử dụng khóa này để kiểm soát quyền truy cập đọc/ghi vào các khối dữ liệu (pages) trong bộ nhớ đệm dùng chung (shared buffer pool). Các khóa này được nhả ra ngay lập tức sau khi hàng dữ liệu được lấy hoặc cập nhật xong, và lập trình viên thường không cần phải bận tâm đến chúng.

**Khóa tư vấn (Advisory Locks)**

- **Khái niệm:** Đây là các khóa do ứng dụng tự định nghĩa ý nghĩa hệ thống, PostgreSQL không tự động can thiệp hay ép buộc. Cơ chế này rất nhẹ, nhanh, tránh làm phình to bảng (table bloat) và thường dùng để mô phỏng chiến lược khóa bi quan.
- **Đặc điểm:** Có hai cấp độ là cấp phiên (session level - tồn tại cho đến khi đóng kết nối hoặc nhả thủ công, không mất đi kể cả khi giao dịch bị rollback) và cấp giao dịch (transaction level - tự động nhả khi hết giao dịch).

## 2. Phân tích các vấn đề và rủi ro tiềm ẩn (Problems & Risks)

**Vấn đề Bế tắc (Deadlocks)**

- **Mô tả:** Việc sử dụng khóa tường minh làm tăng nguy cơ bế tắc. Điều này xảy ra khi hai (hoặc nhiều) giao dịch đang giữ các khóa mà đối phương cần, dẫn đến việc cả hai chờ đợi nhau vô thời hạn. Nó có thể xảy ra ngay cả khi bạn không chủ động khóa mà chỉ dùng lệnh `UPDATE` thông thường trên nhiều hàng (gây ra khóa cấp hàng).
- **Cách giải quyết:** PostgreSQL sẽ tự động phát hiện tình trạng này và hủy (abort) ngẫu nhiên một giao dịch để giao dịch còn lại có thể hoàn thành. Để phòng tránh, lập trình viên phải đảm bảo mọi giao dịch đều khóa các đối tượng theo cùng một thứ tự nhất quán, luôn yêu cầu mức khóa khắt khe nhất ngay từ đầu và không duy trì giao dịch mở quá lâu (ví dụ: không chờ người dùng nhập liệu giữa chừng).

**Chi phí hiệu suất I/O do khóa hàng**

- **Mô tả:** PostgreSQL không lưu trữ thông tin về các hàng bị khóa trong bộ nhớ, do đó không có giới hạn về số lượng hàng có thể bị khóa cùng lúc.
- **Vấn đề:** Để ghi nhớ việc khóa, PostgreSQL phải sửa đổi trực tiếp dữ liệu của các hàng đó để đánh dấu, dẫn đến việc ghi xuống ổ đĩa (disk writes). Nếu bạn chạy `SELECT FOR UPDATE` trên hàng triệu dòng, nó sẽ tạo ra áp lực ghi (I/O) cực lớn lên ổ đĩa.

**Cạn kiệt bộ nhớ quản lý khóa (Exhaustion of Lock Memory)**

- **Mô tả:** Mặc dù khóa hàng ghi trực tiếp xuống đĩa, nhưng cả khóa bảng thông thường và khóa tư vấn (advisory locks) đều được lưu trong một vùng bộ nhớ dùng chung (shared memory pool). Kích thước vùng nhớ này được cấu hình tĩnh thông qua thông số `max_locks_per_transaction` và `max_connections`.
- **Vấn đề:** Nếu ứng dụng tạo ra quá nhiều khóa (vài chục nghìn đến hàng trăm nghìn khóa) làm cạn kiệt vùng nhớ này, PostgreSQL sẽ từ chối cấp phát thêm bất kỳ khóa nào mới, gây gián đoạn toàn bộ hệ thống.

**Khóa treo (Dangling Locks) với Advisory Locks**

- **Mô tả:** Khi kết hợp khóa tư vấn trong các câu lệnh SQL có mệnh đề sắp xếp (`ORDER BY`) hoặc giới hạn (`LIMIT`), thứ tự đánh giá biểu thức của SQL có thể tạo ra lỗi logic.
- **Vấn đề:** Mệnh đề `LIMIT` không được đảm bảo sẽ chạy trước hàm cấp khóa. Ứng dụng có thể vô tình lấy ra và khóa rất nhiều bản ghi ngoài ý muốn, sau đó không thể tự động nhả các khóa này (để lại các khóa treo lơ lửng cho đến khi hết phiên).

Nguồn: https://www.postgresql.org/docs/current/explicit-locking.html


Là Heap Table

Có [[DE/Databases/RDBMS/MySQL/Architecture|Architecture]]

Dữ liệu được lưu ngẫu nhiên trên disk

- Khi INSERT, dữ liệu được thêm vào chỗ trống bất kì trên disk

- Khi DELETE dữ liệu sẽ chưa được xóa ngay mà phải chờ được dọn dẹp ([[Vacuuming]], ...) 

- Khi UPDATE dữ liệu cũ sẽ được giữ lại (do cơ chế MVCC) và dữ liệu mới sẽ được ghi vào dẫn đến tình trạng [[Heap-Only-Tuples (HOT)]] trên toàn bộ cây index và phìng to dữ liệu

Dữ liệu khi được INSERT rất nhanh nhưng đổi lại khi tìm kiếm mà không có index thì rất chậm

- Khi tìm kiếm dữ liệu, postgreSQL phải tìm kiếm toàn bảng khi không có index

Không hỗ trợ nhiều storage engine

Hoạt động dựa trên cơ chế MVCC mạnh mẽ, cung cấp cho các bản ghi một ctid

Mỗi 1 bảng (không phân vùng) bao gồm 1 file data và các file index riêng

Các sự kiện thay đổi dữ liệu cũng như cơ chế write ahead log được gọi là [[WAL]]

Cung cấp các [[DE/Databases/RDBMS/PostgreSQL/Isolation levels|Isolation levels]] 

Cung cấp các [[DE/Databases/RDBMS/PostgreSQL/Indexes|Indexes]]
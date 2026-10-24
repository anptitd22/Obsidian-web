---
title: Glossary
tags:
  - MySQL
---
## Phân bố dữ liệu
### [row](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_row)

Cấu trúc dữ liệu logic được xác định bởi một tập hợp các cột. Một tập hợp các hàng tạo nên một bảng. Trong các tệp dữ liệu InnoDB, mỗi trang có thể chứa một hoặc nhiều hàng.

Mặc dù InnoDB sử dụng thuật ngữ **row format** để nhất quán với cú pháp MySQL, nhưng định dạng hàng là một thuộc tính của mỗi bảng và áp dụng cho tất cả các hàng trong bảng đó.

See Also [column](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_column), [data files](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_data_files), [page](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_page), [row format](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_row_format).

### [row lock](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_row)

Khóa này ngăn chặn việc truy cập một hàng theo cách không tương thích bởi một giao dịch khác. Các hàng khác trong cùng bảng vẫn có thể được ghi tự do bởi các giao dịch khác. Đây là loại khóa được thực hiện bởi các thao tác DML trên các bảng InnoDB.

Ngược lại với các khóa bảng được sử dụng bởi [`MyISAM`](https://dev.mysql.com/doc/refman/9.7/en/myisam-storage-engine.html "18.2 The MyISAM Storage Engine"), hoặc trong các thao tác DDL trên các bảng InnoDB mà không thể thực hiện bằng DDL trực tuyến; các khóa đó ngăn chặn việc truy cập đồng thời vào bảng.

See Also [DDL](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_ddl), [DML](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_dml), [InnoDB](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_innodb), [lock](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_lock), [locking](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_locking), [online DDL](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_online_ddl), [table lock](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_table_lock), [transaction](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_transaction).

### [row-level locking](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_row)

Cơ chế khóa được sử dụng cho các bảng InnoDB dựa trên khóa hàng thay vì khóa bảng. Nhiều giao dịch có thể sửa đổi cùng một bảng đồng thời. Chỉ khi hai giao dịch cố gắng sửa đổi cùng một hàng thì một trong hai giao dịch mới chờ giao dịch kia hoàn thành (và giải phóng khóa hàng của nó).

See Also [InnoDB](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_innodb), [locking](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_locking), [row lock](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_row_lock), [table lock](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_table_lock), [transaction](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_transaction).

### [page](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_page)

Một trang là đơn vị biểu thị lượng dữ liệu mà InnoDB truyền tải tại một thời điểm giữa đĩa (các tập dữ liệu) và bộ nhớ (vùng đệm). Một trang có thể chứa một hoặc nhiều hàng, tùy thuộc vào lượng dữ liệu trong mỗi hàng. Nếu một hàng không nằm gọn hoàn toàn trong một trang, InnoDB sẽ thiết lập các cấu trúc dữ liệu kiểu con trỏ bổ sung để thông tin về hàng đó có thể được lưu trữ trong một trang.

Một cách để chứa nhiều dữ liệu hơn trên mỗi trang là sử dụng định dạng hàng nén. Đối với các bảng sử dụng BLOB hoặc các trường văn bản lớn, định dạng hàng nén cho phép các cột lớn đó được lưu trữ riêng biệt với phần còn lại của hàng, giảm chi phí I/O và mức sử dụng bộ nhớ cho các truy vấn không tham chiếu đến các cột đó.

Khi InnoDB đọc hoặc ghi các tập trang theo lô để tăng thông lượng I/O, nó sẽ đọc hoặc ghi từng phần dữ liệu một.

Tất cả các cấu trúc dữ liệu đĩa InnoDB trong một phiên bản MySQL đều có cùng kích thước trang.

See Also [buffer pool](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_buffer_pool), [data files](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_data_files), [extent](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_extent), [page size](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_page_size), [row](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_row).

### [page size](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_page)

Đối với các phiên bản MySQL từ 5.5 trở xuống, kích thước của mỗi trang InnoDB được cố định ở mức 16 kilobyte. Giá trị này thể hiện sự cân bằng: đủ lớn để chứa dữ liệu cho hầu hết các hàng, nhưng đủ nhỏ để giảm thiểu chi phí hiệu năng do việc chuyển dữ liệu không cần thiết vào bộ nhớ. Các giá trị khác chưa được kiểm thử hoặc hỗ trợ.

Bắt đầu từ MySQL 5.6, kích thước trang cho một phiên bản InnoDB có thể là 4KB, 8KB hoặc 16KB, được điều khiển bởi tùy chọn cấu hình innodb_page_size. Từ MySQL 5.7.6 trở đi, InnoDB cũng hỗ trợ kích thước trang 32KB và 64KB. Đối với kích thước trang 32KB và 64KB, ROW_FORMAT=COMPRESSED không được hỗ trợ và kích thước bản ghi tối đa là 16KB.

Kích thước trang được thiết lập khi tạo phiên bản MySQL và sẽ không thay đổi sau đó. Kích thước trang này áp dụng cho tất cả các tablespace của InnoDB, bao gồm tablespace hệ thống, tablespace file-per-table và tablespace chung.

Kích thước trang nhỏ hơn có thể giúp cải thiện hiệu suất đối với các thiết bị lưu trữ sử dụng kích thước khối nhỏ, đặc biệt là đối với các thiết bị SSD trong các tác vụ phụ thuộc vào ổ đĩa, chẳng hạn như các ứng dụng OLTP. Khi các hàng riêng lẻ được cập nhật, ít dữ liệu hơn được sao chép vào bộ nhớ, ghi vào đĩa, sắp xếp lại, khóa, v.v.

See Also [disk-bound](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_disk_bound), [file-per-table](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_file_per_table), [general tablespace](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_general_tablespace), [instance](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_instance), [OLTP](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_oltp), [page](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_page), [SSD](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_ssd), [tablespace](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_tablespace).

###  [extent](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_extent)

Một nhóm các trang trong một tablespace. Với kích thước trang mặc định là 16KB, một extent chứa 64 trang. Trong MySQL 5.6, kích thước trang cho một instance InnoDB có thể là 4KB, 8KB hoặc 16KB, được điều khiển bởi tùy chọn cấu hình  [`innodb_page_size`](https://dev.mysql.com/doc/refman/9.7/en/innodb-parameters.html#sysvar_innodb_page_size). Đối với kích thước trang 4KB, 8KB và 16KB, kích thước extent luôn là 1MB (hoặc 1048576 byte).

Trong MySQL 5.7.6, hỗ trợ kích thước trang InnoDB 32KB và 64KB đã được thêm vào. Đối với kích thước trang 32KB, kích thước vùng nhớ extent là 2MB. Đối với kích thước trang 64KB, kích thước vùng nhớ extent là 4MB.

Các tính năng của InnoDB như **segments**, yêu cầu **read-ahead** và **doublewrite buffer** sử dụng các thao tác I/O để đọc, ghi, cấp phát hoặc giải phóng dữ liệu từng phần một.

See Also [doublewrite buffer](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_doublewrite_buffer), [page](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_page), [page size](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_page_size), [read-ahead](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_read_ahead), [segment](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_segment), [tablespace](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_tablespace).

### [data files](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_data_files)

Các tệp chứa dữ liệu **table** and **index** data.

The `InnoDB` **system tablespace**, nơi lưu trữ `InnoDB` **data dictionary** và có khả năng chứa dữ liệu cho nhiều bảng InnoDB, được biểu diễn bằng một hoặc nhiều tệp dữ liệu .ibdata.

Các tablespace "file-per-table" (mỗi tablespace chứa dữ liệu cho một bảng InnoDB duy nhất) được biểu diễn bằng tệp dữ liệu .ibd.

Không gian bảng tổng quát (được giới thiệu trong MySQL 5.7.6), có thể chứa dữ liệu cho nhiều bảng InnoDB, cũng được biểu diễn bằng tệp dữ liệu .ibd.

See Also [file-per-table](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_file_per_table), [general tablespace](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_general_tablespace), [ibdata file](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_ibdata_file), [index](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_index), [tablespace](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_tablespace).

### [segment](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_segment)

Một segment bên trong một tablespace của InnoDB. Nếu một tablespace tương tự như một thư mục, thì các segment tương tự như các tập tin bên trong thư mục đó. Một segment có thể mở rộng. Các segment mới có thể được tạo ra.

Ví dụ, trong một tablespace có cấu trúc "mỗi bảng một tệp", dữ liệu của bảng nằm trong một segment và mỗi chỉ mục liên kết nằm trong một segment riêng biệt. Tablespace hệ thống chứa nhiều segment khác nhau, vì nó có thể chứa nhiều bảng và các chỉ mục liên kết của chúng. Trước MySQL 8.0, tablespace hệ thống cũng bao gồm một hoặc nhiều segment rollback được sử dụng cho undo logs.

Các segment sẽ tăng hoặc giảm dung lượng khi dữ liệu được chèn và xóa. Khi một segment cần thêm dung lượng, nó sẽ được mở rộng thêm một extent (1 megabyte) mỗi lần. Tương tự, một phân đoạn sẽ giải phóng dung lượng tương đương một extent khi tất cả dữ liệu trong extent đó không còn cần thiết nữa.

### [tablespace](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_tablespace)

.....

See Also [data files](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_data_files), [file-per-table](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_file_per_table), [general tablespace](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_general_tablespace), [index](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_index), [innodb_file_per_table](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_innodb_file_per_table).

## CPU & I/O bound

### [CPU-bound]()

Một loại khối lượng công việc mà nút thắt cổ chai chính là các thao tác CPU trong bộ nhớ. Thông thường liên quan đến các thao tác đọc chuyên sâu, trong đó tất cả kết quả có thể được lưu trữ tạm thời trong bộ đệm.

See Also [bottleneck](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_bottleneck), [buffer pool](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_buffer_pool), [workload](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_workload).

### [disk-bound]()

Một loại khối lượng công việc mà nút thắt cổ chai chính là thao tác đọc/ghi dữ liệu từ đĩa (còn được gọi là khối lượng công việc bị giới hạn bởi I/O). Thường liên quan đến việc ghi dữ liệu thường xuyên vào đĩa, hoặc đọc ngẫu nhiên lượng dữ liệu lớn hơn dung lượng bộ đệm có thể chứa.

See Also [bottleneck](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_bottleneck), [buffer pool](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_buffer_pool), [workload](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_workload).

## Thiết kế

### [denormalized](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_denormalized)

Một chiến lược lưu trữ dữ liệu duplicates dữ liệu trên các bảng khác nhau, thay vì liên kết các bảng bằng  **foreign keys** và **join** queries. Thường được sử dụng trong các ứng dụng **data warehouse**, nơi dữ liệu không được cập nhật sau khi tải. Trong các ứng dụng như vậy, hiệu suất truy vấn quan trọng hơn việc đơn giản hóa việc duy trì tính nhất quán của dữ liệu trong quá trình cập nhật. Ngược lại với dữ liệu được chuẩn hóa.

See Also [data warehouse](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_data_warehouse), [foreign key](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_foreign_key), [join](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_join), [normalized](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_normalized).

### [normalized](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_normalized)

Chiến lược thiết kế cơ sở dữ liệu trong đó dữ liệu được chia thành nhiều bảng, và các giá trị trùng lặp được gộp lại thành một hàng duy nhất được biểu thị bằng một ID, nhằm tránh việc lưu trữ, truy vấn và cập nhật các giá trị dư thừa hoặc dài dòng. Chiến lược này thường được sử dụng trong các ứng dụng OLTP .

...

See Also [denormalized](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_denormalized), [foreign key](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_foreign_key), [OLTP](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_oltp), [relational](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_relational).

### buffer

Vùng nhớ hoặc vùng đĩa được sử dụng để lưu trữ tạm thời. Dữ liệu được đệm trong bộ nhớ để có thể ghi vào đĩa một cách hiệu quả, với một vài thao tác I/O lớn thay vì nhiều thao tác nhỏ. Dữ liệu được đệm trên đĩa để tăng độ tin cậy, sao cho có thể khôi phục ngay cả khi xảy ra sự cố hoặc lỗi khác vào thời điểm tồi tệ nhất. Các loại bộ đệm chính được InnoDB sử dụng là buffer pool, doublewrite buffer và change buffer.

Nguồn: [[MySQL  MySQL 9.7 Reference Manual  MySQL Glossary]]

### Shared lock

Một loại khóa cho phép các giao dịch khác đọc đối tượng bị khóa và cũng có thể giành được các shared locks khác trên đối tượng đó, nhưng không được phép ghi vào đối tượng. Ngược lại với khóa độc quyền.

See Also [exclusive lock](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_exclusive_lock), [lock](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_lock), [transaction](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_transaction).

### exclusive lock

Một loại khóa ngăn chặn bất kỳ giao dịch nào khác khóa cùng một hàng. Tùy thuộc vào **isolation level**, loại khóa này có thể chặn các giao dịch khác ghi vào cùng một hàng, hoặc cũng có thể chặn các giao dịch khác đọc cùng một hàng. Mức độ cách ly mặc định của InnoDB, REPEATABLE READ, cho phép đồng thời cao hơn bằng cách cho phép các giao dịch đọc các hàng có khóa độc quyền, một kỹ thuật được gọi là đọc nhất quán (**consistent read**).

See Also [concurrency](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_concurrency), [consistent read](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_consistent_read), [isolation level](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_isolation_level), [lock](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_lock), [REPEATABLE READ](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_repeatable_read), [shared lock](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_shared_lock), [transaction](https://dev.mysql.com/doc/refman/9.7/en/glossary.html#glos_transaction).
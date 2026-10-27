- **Định nghĩa**: Là tập hợp dữ liệu phân tán (được thêm từ Spark 1.6).

- **Ưu điểm**: Kết hợp ưu điểm của **RDD** (kiểu dữ liệu mạnh - strong typing, hỗ trợ hàm lambda) với engine tính toán tối ưu của **Spark SQL**.

- **Khởi tạo & Thao tác**: Được tạo từ các đối tượng JVM và biến đổi qua các hàm chức năng (`map`, `flatMap`, `filter`...).

- **Hỗ trợ ngôn ngữ**: Có sẵn trong **Scala** và **Java**. _(Python và R không có Dataset API trực tiếp do tính chất ngôn ngữ động, nhưng vẫn truy cập trường dữ liệu theo tên dạng_ __row.columnName__).


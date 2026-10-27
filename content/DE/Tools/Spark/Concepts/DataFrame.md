- **Định nghĩa**: Là một **Dataset được tổ chức thành các cột có tên** (named columns).

- **Bản chất**: Tương đương với một bảng (table) trong CSDL quan hệ hoặc Dataframe trong R/Python, nhưng được tối ưu hóa mạnh mẽ hơn bên dưới.

- **Nguồn dữ liệu tạo thành**: Tạo từ tệp dữ liệu có cấu trúc, bảng Hive, CSDL bên ngoài hoặc từ RDD có sẵn.

- **Hỗ trợ ngôn ngữ**: Khả dụng trên cả **Scala, Java, Python và R**.

- **Biểu diễn trong Scala/Java**:
    - Trong **Scala**: `DataFrame` là một định danh kiểu (type alias) của `Dataset[Row]`.
    - Trong **Java**: Được biểu diễn trực tiếp bằng `Dataset<Row>`.
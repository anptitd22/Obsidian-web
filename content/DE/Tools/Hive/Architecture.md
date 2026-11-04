![[Pasted image 20261104005408.png]]

```
User Query (HiveQL)
    ↓
Hive Compiler & Optimizer
    ↓
Execution Engine (MapReduce/Tez/Spark)
    ↓
Hadoop YARN
    ↓
HDFS (Data Storage)
```

## 1. Các thành phần chính (Major Components)

- **UI (User Interface)**: Giao diện cho phép người dùng gửi các câu lệnh truy vấn và thao tác tới hệ thống (gồm giao diện dòng lệnh CLI và Web GUI).
- **Driver**: Thành phần tiếp nhận các truy vấn, quản lý session handles và cung cấp các API `execute`, `fetch` dựa trên chuẩn giao tiếp JDBC/ODBC.
- **Compiler**: Nhận câu lệnh truy vấn, phân tích cú pháp (parse), kiểm tra ngữ nghĩa (semantic analysis) và phối hợp với Metastore để tạo Kế hoạch thực thi (Execution Plan).
- **Metastore**: Kho lưu trữ toàn bộ thông tin cấu trúc (metadata) của bảng, phân vùng (partition), kiểu cột, thông tin SerDe (Serializer/Deserializer) và đường dẫn file lưu dữ liệu trên HDFS.
- **Execution Engine**: Thành phần quản lý và thực thi kế hoạch truy vấn được tạo bởi Compiler dưới dạng một đồ thị DAG gồm các giai đoạn (stages) phụ thuộc nhau.

## 2. Quy trình xử lý một câu truy vấn (Query Flow)

```
[UI] --(1. Execute)--> [Driver] --(2. Gửi Query)--> [Compiler]
                                                        │
                      [Metastore] <--(3 & 4. Metadata)──┘
                                                        │
[Driver] <--------------------(5. Kế hoạch DAG)─────────┘
   │
[Execution Engine] <--(6. Thực thi)
   │
   ├── (6.1, 6.2, 6.3) Gửi MapReduce Jobs / HDFS tasks
   └── (7, 8, 9) Đọc kết quả từ File tạm trên HDFS -> Trả về Driver -> UI
```

### Các bước chi tiết:

- **Khởi tạo:** UI gọi hàm execute gửi truy vấn đến Driver. Driver tạo session handle và chuyển query sang Compiler.
    
- **Biên dịch & Metadata:** Compiler tra cứu metadata từ Metastore để:
    
    - Kiểm tra kiểu dữ liệu (Type-checking) của biểu thức.
        
    - Cắt tỉa phân vùng (**Partition pruning**) dựa vào mệnh đề `WHERE`.
        
- **Sinh kế hoạch:** Compiler trả về một DAG gồm các công đoạn (Stages): MapReduce task, metadata operation hoặc HDFS task.
    
- **Thực thi:**
    
    - Execution Engine phân phối các tác vụ MapReduce.
        
    - **Mapper/Reducer** dùng **Deserializer** để giải mã dòng dữ liệu từ HDFS vào cây toán tử (Operator tree).
        
    - Dữ liệu đầu ra được ghi ra file tạm trên HDFS qua **Serializer**.
        
- **Đảm bảo tính toàn vẹn:**
    
    - **Truy vấn đọc (SELECT):** Đọc trực tiếp từ file tạm HDFS trả về cho Driver thông qua lời gọi fetch.
        
    - **Truy vấn ghi (DML - INSERT):** File tạm sẽ được đổi tên (**Atomic rename**) về thư mục chính thức của bảng. Cơ chế này ngăn chặn việc đọc dữ liệu rác (dirty read).

## 3. Mô hình dữ liệu trong Hive (Data Model)

### 3.1. Cấu trúc lưu trữ

- **Tables (Bảng):**
    
    - Tương đương bảng trong RDBMS (lọc, gom nhóm, join, union).
        
    - Ứng với **một thư mục trên HDFS**.
        
    - Hỗ trợ **External Table**: Tạo bảng trỏ trực tiếp đến file/thư mục HDFS có sẵn mà không can thiệp xóa dữ liệu gốc khi drop bảng.
        
- **Partitions (Phân vùng):**
    
    - Chia dữ liệu dựa trên một hoặc nhiều partition key (ví dụ: `ds='2008-09-01'`).
        
    - Ứng với **thư mục con** theo định dạng: `<table_dir>/ds=2008-09-01/`.
        
    - **Lợi ích:** Tránh full scan toàn bảng nhờ cơ chế **Partition Pruning**.
        
- **Buckets (Thùng dữ liệu):**
    
    - Dữ liệu trong partition tiếp tục được băm (**hash**) theo một cột cụ thể.
        
    - Mỗi bucket ứng với **một file riêng lẻ** bên trong thư mục partition.
        
    - **Lợi ích:** Tối ưu hóa truy vấn lấy mẫu ngẫu nhiên (`SAMPLE`) và tăng tốc phép toán Join.
        

### 3.2. Hệ thống kiểu dữ liệu & Cơ chế SerDe

- **Kiểu cơ bản (Primitive):** `int`, `float`, `string`, `date`, `boolean`.
    
- **Kiểu tập hợp (Collections):** `array`, `map` (hỗ trợ hàm tích hợp như `size()`, toán tử lấy chỉ mục).
    
- **Kiểu lồng nhau (Nested Types):** Dùng cú pháp dấu chấm (`a.b.c = 1`).
    
- **Cơ chế SerDe & ObjectInspector:**
    
    - **SerDe (Serializer / Deserializer):** Cho phép Hive đọc/ghi dữ liệu ở bất kỳ định dạng file nào trên HDFS.
        
    - **ObjectInspector:** Hướng dẫn Hive cách bóc tách cấu trúc đối tượng (hỗ trợ sẵn `ListObjectInspector`, `StructObjectInspector`, `MapObjectInspector`).
        
    - Người dùng có thể tự viết SerDe và ObjectInspector để hỗ trợ kiểu dữ liệu tùy chỉnh.

## 4. Kiến trúc Metastore

### 4.1. Vai trò

- **Data Abstraction (Trừu tượng hóa dữ liệu):** Định nghĩa cấu trúc, định dạng file, SerDe một lần khi tạo bảng; tái sử dụng trong mọi truy vấn sau đó (không cần khai báo lại schema khi truy vấn).
    
- **Data Discovery (Khám phá dữ liệu):** Lưu tập trung metadata, cho phép người dùng và công cụ bên ngoài khám phá kho dữ liệu một cách đồng bộ.
    

### 4.2. Các thực thể Metadata (Objects)

- **Database:** Không gian tên (namespace) nhóm các bảng lại; mặc định là database `default`.
    
- **Table:** Chứa schema cột, chủ sở hữu, thông tin lưu trữ HDFS, định dạng Input/Output, SerDe class, thông tin Bucketing.
    
- **Partition:** Mỗi phân vùng có thể lưu trữ schema, SerDe và đường dẫn HDFS riêng, giúp thay đổi schema của bảng mà không phá vỡ dữ liệu các phân vùng cũ.
    

### 4.3. Kiến trúc kỹ thuật & Chế độ triển khai

- **Backend Storage:** Lưu trong CSDL quan hệ (RDBMS) thông qua tầng **DataNucleus ORM**.
    
    - _Lý do không lưu metadata trên HDFS:_ HDFS không hỗ trợ cập nhật ngẫu nhiên (random update) và metadata cần khả năng truy vấn nhanh (queriability).
        
- **Các chế độ triển khai (Deployment Modes):**
    
    - **Embedded Mode:** Hive Client kết nối trực tiếp đến RDBMS qua JDBC (dễ dùng, ít service cần giám sát).
        
    - **Remote Mode:** Metastore chạy như một **Thrift Service độc lập**, client kết nối qua mạng (tốt cho client không phải Java hoặc kiến trúc đa người dùng).
        
    - **Local Mode:** Biến thể kết nối trực tiếp trong môi trường nội bộ.

## 5. Ngôn ngữ truy vấn HiveQL

- **Đặc tính:** Cú pháp tương tự SQL (DDL, DML, SELECT).
    
- **Tích hợp Custom MapReduce Scripts:**
    
    - Cho phép nhúng mã nguồn tùy biến bằng bất kỳ ngôn ngữ nào qua cơ chế **Streaming (stdin / stdout)**.
        
    - Đánh đổi: Tốn chi phí ép kiểu dữ liệu từ/sang dạng chuỗi (strings).
        
- **Multi-table Insert:**
    
    - Thực hiện ghi dữ liệu vào nhiều bảng đích khác nhau chỉ bằng một lần scan nguồn vào:
        
        SQL
        
        ```
        FROM source_table
        INSERT OVERWRITE TABLE target1 SELECT ...
        INSERT OVERWRITE TABLE target2 SELECT ...;
        ```
        
    - Tăng hiệu năng gấp nhiều lần nhờ tái sử dụng một lượt quét dữ liệu.

## 6. Trình biên dịch & Tối ưu hóa (Compiler & Optimizer)

### 6.1. Các giai đoạn biên dịch

1. **Parser:** Chuyển chuỗi query thành cây cú pháp trừu tượng (**Parse Tree**).
    
2. **Semantic Analyser:**
    
    - Chuyển parse tree thành biểu diễn dạng khối (Internal Query Representation).
        
    - Kiểm tra tên cột, bung toán tử `*`, type-checking, gom các điều kiện lọc để phục vụ tỉa phân vùng (partition pruning) và lấy mẫu (sampling).
        
3. **Logical Plan Generator:**
    
    - Sinh cây toán tử logic (Logical Plan) gồm các toán tử đại số quan hệ (`filter`, `join`...).
        
    - Xuất hiện toán tử ranh giới **`reduceSink`** (nằm giữa Map và Reduce, chứa các key phục vụ bước shuffle/sort).
        
    - Tối ưu hóa sơ bộ: Gộp nhiều join thành multi-way join, partial aggregation ở tầng Map cho `GROUP BY`, tách `GROUP BY` 2 pha chống lệch dữ liệu (skewed data).
        
4. **Query Plan Generator:**
    
    - Đệ quy duyệt cây toán tử logic và cắt thành các tác vụ MapReduce/HDFS có thể tuần tự hóa (serializable). Kế hoạch cuối cùng được ghi thành file để chuẩn bị thực thi.
        

### 6.2. Các kỹ thuật tối ưu hóa (Optimizer)

- **Rule-based Optimization:**
    
    - **Column Pruning:** Chỉ đọc đúng các cột được chỉ định trong truy vấn.
        
    - **Predicate Pushdown:** Đẩy điều kiện lọc xuống sâu nhất có thể (lọc ngay tại tầng đọc dữ liệu).
        
- **Mở rộng nâng cao:**
    
    - **Map-side Join:** Đưa bảng nhỏ vào bộ nhớ của Mapper để tránh bước Shuffle/Reduce tốn kém.
        
    - **Cost-based Optimization (CBO):** Dùng thống kê phân phối dữ liệu (Statistics) để chọn thứ tự join và kế hoạch tối ưu.
        
    - **Correlation Optimizer:** Khai thác các thuộc tính tương đồng giữa các job MapReduce để gộp tác vụ.
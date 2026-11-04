## 1. Bản chất ACID trong Hive & Bài toán giải quyết

### 1.1. Khái niệm cốt lõi

- **A (Atomicity):** Một câu lệnh hoặc batch ghi hoàn tất toàn bộ hoặc thất bại toàn bộ, không để lại dữ liệu dở dang (partial files).
    
- **C (Consistency):** Ngay khi một thao tác ghi thành công, các truy vấn sau đó lập tức nhìn thấy dữ liệu mới.
    
- **I (Isolation):** Các giao dịch đồng thời không gây xung đột dữ liệu ngoài ý muốn. Hive hỗ trợ mức độ **Snapshot Isolation**.
    
- **D (Durability):** Dữ liệu một khi đã commit sẽ được lưu trữ an toàn trên HDFS dù hệ thống gặp sự cố sập node.

### 1.2. Tại sao Data Warehouse trên HDFS lại cần ACID?

Vốn dĩ HDFS là hệ thống file **Append-only**, không hỗ trợ ghi đè tại chỗ (_in-place update_), và việc nhiều tiến trình cùng đọc/ghi một thư mục rất dễ sinh lỗi **Dirty Read** hoặc tạo ra hàng triệu file nhỏ (_small files issue_). ACID ra đời để xử lý 4 bài toán:

1. **Streaming Data Ingest (Kafka, Flume, Storm đổ về liên tục):**
    
    - _Trước ACID:_ Nếu ghi vào phân vùng hiện có, các truy vấn đọc cùng lúc sẽ gặp dirty read. Để tránh điều đó, người ta phải tạo phân vùng mới liên tục (15-60 phút/lần) $\rightarrow$ gây bùng nổ số lượng partition, quá tải NameNode.
        
    - _Với ACID:_ Cho phép stream ghi trực tiếp vào partition đang có mà người đọc vẫn thấy dữ liệu nhất quán (snapshot) và không lo lock toàn bảng.
        
2. **SCD Type 1 & 2 (Slowly Changing Dimensions):** Hỗ trợ cập nhật các chiều dữ liệu biến đổi chậm (ví dụ: khách hàng đổi địa chỉ, cửa hàng thay đổi diện tích) qua các lệnh `UPDATE`, `INSERT`.
    
3. **Data Restatement & GDPR / Data Purging:** Khả năng sửa chữa dữ liệu sai lệch, cập nhật bổ sung số liệu muộn, hoặc xóa thông tin người dùng (`DELETE`) theo yêu cầu pháp lý (GDPR/Right to be Forgotten).
    
4. **Bulk Updates qua `MERGE`:** Đồng bộ dữ liệu hàng loạt từ bảng staging vào bảng chính (UPSERT) chỉ trong một câu lệnh chuẩn SQL.

## 2. Thiết kế tầng lưu trữ: Base và Delta Directories

Vì HDFS không hỗ trợ sửa file trực tiếp, Hive giải quyết bài toán mutation bằng mô hình **LSM-tree-like / Copy-on-Write lai Append-only**:


```
/user/hive/warehouse/table_name/
│
├── base_0000022/                     <-- Dữ liệu gốc đã hợp nhất (đến txn 22)
│   ├── bucket_00000
│   └── bucket_00001
│
├── delta_0000023_0000023_0000/       <-- Ghi nhận thay đổi tại txn 23 (Insert/Update)
│   ├── bucket_00000
│   └── bucket_00001
│
└── delta_0000024_0000024_0000/       <-- Ghi nhận thay đổi tại txn 24
    ├── bucket_00000
    └── bucket_00001
```

- **Thư mục Base (`base_xxx`):** Chứa trạng thái dữ liệu nền tảng đã qua quá trình gộp (compaction) tính đến một mã giao dịch nhất định.
    
- **Thư mục Delta (`delta_xxx_xxx_xxx`):** Mỗi transaction (hoặc batch write) khi thực hiện `INSERT`, `UPDATE`, `DELETE` sẽ tạo ra một bộ file delta mới.
    
    - Cấu trúc tên: `delta_<minTxn>_<maxTxn>_<statementId>`.
        
- **Cơ chế đọc (Read Path):** Reader áp dụng cơ chế **Read-time Merge** — đọc đồng thời thư mục `base` và các thư mục `delta` liên quan, đối chiếu ID bản ghi ngầm/hiện để loại bỏ các dòng bị xóa (`DELETE`) hoặc cập nhật giá trị mới nhất (`UPDATE`).

## 3. Kiến trúc Compactor (Cơ chế dọn dẹp và hợp nhất file)

Càng nhiều câu lệnh DML chạy thì số lượng thư mục `delta` càng tăng cao $\rightarrow$ suy giảm nghiêm trọng hiệu năng truy vấn (do phải mở và merge quá nhiều file) và gây áp lực lên HDFS NameNode. Compactor là một nhóm background daemon chạy trong **Hive Metastore** đảm nhận nhiệm vụ này:

### 3.1. Phân loại Compaction

| **Tiêu chí**              | **Minor Compaction**                                                          | **Major Compaction**                                                                 | **Rebalance Compaction**                                             |
| ------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| **Bản chất**              | Gom nhiều thư mục `delta` nhỏ thành **1 file delta duy nhất** cho mỗi bucket. | Gom toàn bộ các file `delta` vào file `base` để tạo ra **1 file base duy nhất** mới. | Tái cấu trúc phân bổ lại bucket khi số lượng bucketing bị lệch.      |
| **Chi phí tài nguyên**    | Nhẹ, chạy nhanh, chỉ xử lý delta.                                             | Nặng, tốn CPU/I/O vì quét toàn bộ dữ liệu bảng/partition.                            | Rất nặng.                                                            |
| **Cơ chế Lock**           | Cho phép Concurrent Reads & Concurrent Writes.                                | Cho phép Concurrent Reads & Concurrent Writes.                                       | Dùng **Exclusive Write Lock** (chặn toàn bộ ghi đồng thời).          |
| **File cũ sau compactor** | Giữ lại cho tới khi các active readers đọc xong, sau đó Cleaner xóa.          | Giữ lại cho tới khi các active readers đọc xong, sau đó Cleaner xóa.                 | Giữ lại cho tới khi các active readers đọc xong, sau đó Cleaner xóa. |
### 3.2. Bốn thành phần cốt lõi của Compactor

1. **Initiator:** Quét định kỳ Metastore để tìm các bảng/partition vượt ngưỡng cấu hình (threshold) cần compaction và đưa yêu cầu vào hàng đợi.
    
2. **Worker:** Nhận tác vụ từ hàng đợi, khởi tạo một **MapReduce Job** (hoặc Query engine) để thực thi việc hợp nhất file vật lý trên HDFS.
    
3. **Cleaner:** Sau khi compaction xong, kiểm tra xem còn query nào đang đọc các file `delta`/`base` cũ không; nếu không còn ai truy cập, Cleaner sẽ xóa bỏ các file cũ để giải phóng dung lượng HDFS.
    
4. **AcidHouseKeeperService:** Quét tìm các giao dịch quá hạn (không gửi tín hiệu heartbeat trong khoảng thời gian `hive.txn.timeout`) để tự động **ABORT** và giải phóng Lock nhằm tránh deadlock/dangling lock.

## 4. Quản lý Giao dịch & Khóa (Transaction / Lock Manager)

Để hiện thực hóa ACID, Hive loại bỏ cơ chế khóa cũ (In-memory, ZooKeeperLockManager) và thay bằng:

- **`DbTxnManager`:** Lưu toàn bộ trạng thái transaction và lock trực tiếp trong **Metastore RDBMS** (`DbLockManager`). Đảm bảo trạng thái giao dịch bền vững (Durable) ngay cả khi HiveServer2 hoặc Metastore restart.
    
- **Cơ chế Heartbeat:** Client đang chạy query/transaction phải gửi heartbeat định kỳ về Metastore. Nếu quá thời gian quy định mà không có heartbeat, Metastore coi như client đã crash và thu hồi Lock.
    
- **Backoff Retry:** Khi gặp xung đột khóa, `DbLockManager` tăng gấp đôi thời gian chờ sau mỗi lần thử lại (bắt đầu từ 100ms: 100ms $\rightarrow$ 200ms $\rightarrow$ 400ms... trần là `hive.lock.sleep.between.retries`) để giảm áp lực dồn dập vào Metastore DB.
    
- **Tính tương thích ngược (`hive.txn.strict.locking.mode`):**
    
    - _Mặc định (Strict = `true`):_ Các bảng non-ACID khi `INSERT` sẽ lấy **Exclusive Lock** (chặn các `INSERT` khác).
        
    - _Non-strict (`false`):_ Trả lại hành vi truyền thống, `INSERT` vào bảng non-ACID chỉ lấy **Shared Lock**, cho phép ghi đồng thời mà vẫn bảo vệ bảng khỏi thao tác `DROP TABLE`.

## 5. Các giới hạn kỹ thuật cần lưu ý (Limitations)

1. **Auto-commit duy nhất:** Chưa hỗ trợ `BEGIN`, `COMMIT`, `ROLLBACK` tường minh từ phía client; mỗi câu lệnh SQL tự đóng gói thành 1 transaction độc lập.
    
2. **Chỉ hỗ trợ định dạng ORC:** Bắt buộc bảng ACID phải được lưu trữ dưới dạng `STORED AS ORC` (do ORC có cấu trúc cột ngầm lưu trữ Row ID, Transaction ID cho phép theo dõi dòng bị xóa/sửa).
    
3. **Bắt buộc Bucketing:** Bảng ACID bắt buộc phải phân cụm (`CLUSTERED BY (...) INTO n BUCKETS`) để map-reduce engine compactor kiểm soát chính xác từng file dữ liệu tương ứng.
    
4. **Không áp dụng cho External Table:** External Table nằm ngoài phạm vi quản lý của Compactor, do đó không thể gán thuộc tính ACID.
    
5. **Mức độ cô lập (Isolation Level):** Chỉ hỗ trợ duy nhất **Snapshot Isolation**. Không hỗ trợ Dirty Read, Read Committed hay Serializable.
    
6. **Không hỗ trợ câu lệnh `LOAD DATA ...`:** Dữ liệu bắt buộc phải đi qua parser/execution engine (`INSERT`, `MERGE` hoặc Streaming API) để gắn nhãn metadata/transaction, không thể copy file raw vào thư mục bảng.

## 6. Tổng hợp Cấu hình quan trọng (Configuration Cheat Sheet)

### 6.1. Bắt buộc kích hoạt trên Client / HiveServer2

```xml
<!-- Bật tính năng đồng thời -->
<property>
  <name>hive.support.concurrency</name>
  <value>true</value>
</property>

<!-- Bắt buộc dùng DbTxnManager -->
<property>
  <name>hive.txn.manager</name>
  <value>org.apache.hadoop.hive.ql.lockmgr.DbTxnManager</value>
</property>

<!-- Cho phép ghi dynamic partition không giới hạn -->
<property>
  <name>hive.exec.dynamic.partition.mode</name>
  <value>nonstrict</value>
</property>
```

### 6.2. Bắt buộc kích hoạt trên Server (Hive Metastore)

```xml
<!-- Khởi chạy luồng Initiator quét bảng cần compact -->
<property>
  <name>hive.compactor.initiator.on</name>
  <value>true</value>
</property>

<!-- Khởi chạy luồng Cleaner dọn file delta cũ -->
<property>
  <name>hive.compactor.cleaner.on</name>
  <value>true</value>
</property>

<!-- Số worker thread xử lý tác vụ compaction (tối thiểu > 0 trên 1 node Metastore) -->
<property>
  <name>hive.compactor.worker.threads</name>
  <value>4</value>
</property>
```

### 6.3. Các ngưỡng kích hoạt Compaction tự động

- `hive.compactor.delta.num.threshold` (mặc định `10`): Khi một partition có **$\ge$ 10 thư mục delta**, tự động kích hoạt **Minor Compaction**.
    
- `hive.compactor.delta.pct.threshold` (mặc định `0.1` = 10%): Khi tổng kích thước các file delta vượt quá **10%** kích thước file base, tự động kích hoạt **Major Compaction**.
    
- `hive.compactor.abortedtxn.threshold` (mặc định `1000`): Khi có **$\ge$ 1000 giao dịch aborted** trên partition, tự động kích hoạt **Major Compaction** để dọn sạch rác.

## 7. Cú pháp DDL & Vận hành

### 7.1. Khởi tạo bảng chuẩn ACID

```sql
CREATE TABLE users_acid (
    user_id     INT,
    user_name   STRING,
    email       STRING,
    updated_at  TIMESTAMP
)
CLUSTERED BY (user_id) INTO 4 BUCKETS
STORED AS ORC
TBLPROPERTIES (
    'transactional'='true',
    'compactorthreshold.hive.compactor.delta.num.threshold'='5',
    'compactorthreshold.hive.compactor.delta.pct.threshold'='0.3'
);
```

**Cảnh báo bất biến:** Một khi bảng đã thiết lập `'transactional'='true'`, **tuyệt đối không thể chuyển ngược lại** thành non-ACID (`'transactional'='false'`).

### 7.2. Lệnh giám sát và điều khiển thủ công

- **Xem trạng thái giao dịch:**
    
```sql
    SHOW TRANSACTIONS;
    ABORT TRANSACTIONS <txn_id>;
```
    
- **Xem và kiểm tra khóa:**
    
```sql
    SHOW LOCKS;
```
    
- **Giám sát Compaction:**
    
```sql
    SHOW COMPACTIONS;
```
    
- **Ép buộc kích hoạt Compaction thủ công (Trigger Compaction):**
    
```sql
    -- Kích hoạt Minor Compaction
    ALTER TABLE users_acid COMPACT 'minor';
    
    -- Kích hoạt Major Compaction kèm ghi đè cấu hình RAM cho mapper
    ALTER TABLE users_acid COMPACT 'major' 
    WITH OVERWRITE TBLPROPERTIES ("compactor.mapreduce.map.memory.mb"="4096");
```
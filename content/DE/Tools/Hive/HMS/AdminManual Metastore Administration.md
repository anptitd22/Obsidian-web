## 1. Tổng quan về Metastore

- **Chức năng:** Là nơi lưu trữ toàn bộ metadata của bảng, phân vùng, kiểu cột, SerDe và đường dẫn file HDFS.
    
- **Cơ chế lưu trữ:** Metadata được ánh xạ và lưu vào CSDL quan hệ thông qua giải pháp ORM DataNucleus (JPOX).
    
- **Tính chất:** Metastore Server là **Stateless** $\rightarrow$ có thể chạy song song nhiều node để đạt tính sẵn sàng cao (**High Availability - HA**).

## 2. Các mô hình thiết lập Metastore (Deployment Architectures)

Kiến trúc Metastore được chia làm 2 khía cạnh: **Metastore Database** và **Metastore Server**.

```
[Mô hình Embedded / Local]
  Hive Client (chứa luôn Metastore Server Library) 
      └──(JDBC)──> Backend Database (Derby hoặc RDBMS từ xa)

[Mô hình Remote Server]
  Nhiều Hive Clients 
      └──(Thrift RPC)──> Metastore Server độc lập 
                            └──(JDBC)──> Backend RDBMS (MySQL, Postgres...)
```

### 2.1. Phân loại theo Metastore Database (Nơi lưu data)

- **Local/Embedded Database (Apache Derby):**
    
    - _Đặc điểm:_ Mỗi thời điểm chỉ duy nhất **1 process** được kết nối vào CSDL.
        
    - _Mục đích sử dụng:_ Thích hợp cho chạy Unit test; **không dùng cho môi trường Production**.
        
- **Remote Database (RDBMS độc lập):**
    
    - _Đặc điểm:_ Sử dụng hệ CSDL quan hệ độc lập (MySQL, Postgres, Oracle, SQL Server).
        
    - _Mục đích sử dụng:_ **Khuyến nghị cho tất cả các hệ thống Production thực tế**.
        

### 2.2. Phân loại theo Metastore Server (Cách phục vụ truy vấn)

- **Local/Embedded Server:**
    
    - Metastore Server chạy như một thư viện (library) nằm trực tiếp bên trong tiến trình của Hive Client / HiveServer2.
        
    - Từng Client tự mở kết nối JDBC trực tiếp tới CSDL $\rightarrow$ Bắt buộc máy chạy Client phải có quyền mạng truy cập DB và phải có sẵn Driver JDBC trong classpath.
        
- **Remote Server:**
    
    - Metastore Server chạy như một Service độc lập qua giao thức **Thrift** (mặc định port `9083`).
        
    - Tất cả Client gửi truy vấn metadata qua Thrift đến Metastore Server; Metastore Server mới là nơi kết nối JDBC tới DB.
        
    - An toàn hơn, bảo mật thông tin đăng nhập CSDL tốt hơn và giảm tải kết nối trực tiếp vào DB.

## 3. Bảng so sánh nhanh các chế độ cấu hình

| **Chế độ**             | **Metastore Server chạy ở đâu?** | **Metastore Database nằm ở đâu?** | **Tham số hive.metastore.uris** | **Đối tượng sử dụng**        |
| ---------------------- | -------------------------------- | --------------------------------- | ------------------------------- | ---------------------------- |
| **Embedded Metastore** | Nằm trong tiến trình Client      | Derby (file cục bộ)               | Để trống                        | Test nhanh, Unit Test        |
| **Local Metastore**    | Nằm trong tiến trình Client      | RDBMS từ xa (MySQL,...)           | Để trống                        | Môi trường nhỏ / HiveServer2 |
| **Remote Metastore**   | Service riêng (`Thrift`)         | RDBMS từ xa (MySQL,...)           | `thrift://<host>:<port>`        | **Production tiêu chuẩn**    |

## 4. Các tham số cấu hình quan trọng (Core Configuration)

### 4.1. Kết nối CSDL (JDBC / JDO)

- `javax.jdo.option.ConnectionURL`: Chuỗi JDBC kết nối DB (vd: `jdbc:mysql://<host>/<db>?createDatabaseIfNotExist=true` hoặc `jdbc:derby:...`).
    
- `javax.jdo.option.ConnectionDriverName`: Tên class driver JDBC (vd: `com.mysql.jdbc.Driver` hoặc `org.apache.derby.jdbc.EmbeddedDriver`).
    
- `javax.jdo.option.ConnectionUserName` & `javax.jdo.option.ConnectionPassword`: Tài khoản kết nối CSDL.
    
- `datanucleus.autoStartMechanism`: Khuyến nghị cấu hình giá trị `SchemaTable` (theo HIVE-4762).

### 4.2. Kết nối và định danh Metastore

- `hive.metastore.uris`: Danh sách URI các Thrift Metastore server (phân cách bằng dấu phẩy).
    
    - _Cơ chế HA:_ Hive ưu tiên kết nối URI đầu tiên; nếu lỗi mạng sẽ tự động random chọn node khác để thử lại.
        
    - _Nhận biết chế độ:_ Nếu để trống $\rightarrow$ Chế độ Local; nếu có giá trị $\rightarrow$ Chế độ Remote (từ bản Hive 0.10 trở đi đã bỏ `hive.metastore.local`).
        
- `hive.metastore.warehouse.dir`: Đường dẫn thư mục gốc trên HDFS chứa dữ liệu các bảng nội bộ (Managed/Native tables).
    
- `hive.metastore.port`: Port dịch vụ Metastore (mặc định `9083`).
    

### 4.3. Quản lý luồng (Threads & Retries)

- `hive.metastore.server.min.threads` / `hive.metastore.server.max.threads`: Số worker thread tối thiểu (mặc định 200) và tối đa (mặc định 100.000) trong thread pool của Thrift server.
    
- `hive.hmshandler.retry.attempts` / `hive.hmshandler.retry.interval`: Số lần thử lại và khoảng thời gian chờ (ms) khi gặp lỗi kết nối tới Datastore.

## 5. Lệnh vận hành Metastore Server

- **Khởi động dịch vụ:**
    
```bash
hive --service metastore
```
    
- **Chỉ định cổng lắng nghe tùy chỉnh:**
    
```bash
hive --service metastore -p <port_num>
```
    
- **Lưu ý triển khai MySQL:** Bắt buộc phải copy file jar MySQL JDBC driver vào thư mục `$HIVE_HOME/lib` trước khi khởi động Client hoặc Server.

## 6. Tính năng nâng cao theo phiên bản

### 6.1. Dynamic Service Discovery qua Apache ZooKeeper (Hive 4.0.0+)

Cho phép Client tự động phát hiện các instance Metastore Server đang hoạt động thông qua ZooKeeper:

- `hive.metastore.service.discovery.mode`: Đặt là `zookeeper`.
    
- `hive.metastore.uris`: Trở thành danh sách địa chỉ của cụm ZooKeeper (dạng `<zk_host>:<zk_port>`).
    
- `hive.metastore.zookeeper.namespace`: Đường dẫn node cha trên ZooKeeper lưu thông tin Metastore.

### 6.2. Tính nhất quán của Schema & Nâng cấp (Schema Verification - Hive 0.12+)

- **Cơ chế:** Hive lưu lại `schema version` trong DB và đối chiếu trực tiếp với phiên bản binary đang chạy để ngăn chặn sai lệch cấu trúc dữ liệu.
    
- **Mặc định:** Hive **không** tự động tạo hay sửa đổi cấu trúc schema khi chạy query (tránh rủi ro ghi đè dữ liệu metadata). Nếu schema không khớp, query sẽ báo lỗi ngay lập tức.
    
- **Tham số ghi đè (không khuyến khích):** Đặt `hive.metastore.schema.verification = false` nếu muốn bỏ qua bước xác thực này.
    
- **Công cụ:** Từ Hive 0.12, cung cấp công cụ ngoại tuyến (**Offline Schema Tool / `schematool`**) để chủ động khởi tạo và nâng cấp schema.

## 7. CSDL Backend được hỗ trợ & Phiên bản tối thiểu

| **Hệ quản trị CSDL** | **Phiên bản tối thiểu** | **Giá trị tham số** |
| -------------------- | ----------------------- | ------------------- |
| **MySQL**            | 5.6.17                  | `mysql`             |
| **PostgreSQL**       | 9.1.13                  | `postgres`          |
| **Oracle**           | 11g                     | `oracle`            |
| **MS SQL Server**    | 2008 R2                 | `mssql`             |

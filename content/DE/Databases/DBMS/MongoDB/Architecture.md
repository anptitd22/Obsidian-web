---
title: Architecture
tags:
  - MongoDB
  - RDBMS
---
## Architecure

![[Pasted image 20261024131010.png]]

## Components

![[Pasted image 20261024130906.png]]

### Người Dùng (Users / Clients)

- **Web/App Users**: Người dùng cuối tương tác qua giao diện.
    
- **Developers**: Lập trình viên truy xuất/ghi dữ liệu qua API, code.
    
- **Business Analysts (BA/DA)**: Phân tích số liệu, chạy báo cáo thống kê.
    
- **Database Administrators (DBA)**: Quản trị, giám sát log, phân quyền, tối ưu hiệu năng.

### Bộ Xử Lý Truy Vấn (Query Processor)

- **Phân tích cú pháp & Diễn dịch (Parsing & Interpretation)**:
    
    - Kiểm tra lỗi cú pháp (Syntax errors).
        
    - Chuyển đổi câu truy vấn dạng văn bản thành định dạng cấu trúc nội bộ (Internal Tree/Graph).
        
- **Tối ưu hóa & Thực thi (Optimization & Execution)**:
    
    - Tạo ra kế hoạch thực thi tối ưu nhất (Best Execution Plan).
        
    - Cân bằng giữa thời gian xử lý và mức tiêu thụ tài nguyên (CPU, I/O, RAM).
        
- **Sử dụng chỉ mục (Indexing)**: Tận dụng các chỉ mục phù hợp để tăng tốc độ quét và lọc dữ liệu.

### Trình Quản Lý Lưu Trữ (Storage Manager)

Bao gồm các hệ thống con cốt lõi:

- **Quản lý vùng đệm (Buffer Manager)**: Điều phối Buffer Pool (RAM), giữ lại dữ liệu hay dùng, giảm thiểu thao tác đọc/ghi trực tiếp xuống đĩa (Disk I/O).
    
- **Quản lý giao dịch (Transaction Manager)**: Đảm bảo kiểm soát đồng thời (Concurrency Control) và cơ chế phục hồi hệ thống khi có sự cố (Recovery / ACID).
    
- **Quản lý tệp (File Manager)**: Quản lý vị trí lưu trữ và truy xuất trên đĩa vật lý, cấp phát/thu hồi không gian đĩa, nén dữ liệu và duy trì tệp chỉ mục.
    
- **Xác thực & Toàn vẹn (Authorization & Integrity Manager)**: Kiểm tra quyền truy cập (Access Control) và bảo vệ các ràng buộc toàn vẹn dữ liệu.

### Bộ Nhớ Vật Lý (Physical Storage)

- Nơi lưu trữ thực tế trên thiết bị phần cứng (HDD/SSD/SAN).
    
- Chứa: **Data files**, **Indexes**, **System Catalogs/Statistics**, **Transaction Logs (WAL/Redo Log)**, và **Cached Plans**.

## SQL & NoSQL

![[Pasted image 20261024131606.png]]

## CÁC MÔ HÌNH KIẾN TRÚC CƠ SỞ DỮ LIỆU (DATABASE ARCHITECTURE MODELS)

### Hai Góc Nhìn Phân Loại Kiến Trúc Cốt Lõi

- **Mô hình Logic (Logical Architecture):**
    
    - Tập trung vào biểu diễn trừu tượng của dữ liệu: cấu trúc, thuộc tính và mối quan hệ giữa các thực thể.
        
    - _Ví dụ tiêu biểu:_ Mô hình quan hệ (Relational), mô hình tài liệu (Document - MongoDB), mô hình phân cấp (Hierarchical), hướng đối tượng (Object-Oriented), mô hình đồ thị (Graph).
        
- **Mô hình Vật lý / Phân tầng (Physical / Tier Architecture):**
    
    - Định nghĩa cách hệ thống được triển khai thực tế trên hạ tầng phần cứng, mạng và máy chủ.
        
    - Phân chia vai trò xử lý giữa phía yêu cầu (Client) và phía phản hồi/lưu trữ (Server).

### Các Mô Hình Kiến Trúc Phân Tầng Vật Lý (Tier Architecture)

## 1. Hai Góc Nhìn Phân Loại Kiến Trúc Cốt Lõi

- **Mô hình Logic (Logical Architecture):**
    
    - Tập trung vào biểu diễn trừu tượng của dữ liệu: cấu trúc, thuộc tính và mối quan hệ giữa các thực thể.
        
    - _Ví dụ tiêu biểu:_ Mô hình quan hệ (Relational), mô hình tài liệu (Document - MongoDB), mô hình phân cấp (Hierarchical), hướng đối tượng (Object-Oriented), mô hình đồ thị (Graph).
        
- **Mô hình Vật lý / Phân tầng (Physical / Tier Architecture):**
    
    - Định nghĩa cách hệ thống được triển khai thực tế trên hạ tầng phần cứng, mạng và máy chủ.
        
    - Phân chia vai trò xử lý giữa phía yêu cầu (Client) và phía phản hồi/lưu trữ (Server).
        

### 2. Các Mô Hình Kiến Trúc Phân Tầng Vật Lý (Tier Architecture)

![[Pasted image 20261024131939.png]]

#### 2.1. Kiến Trúc 1 Tầng (1-Tier Architecture)

- **Cấu trúc:** Toàn bộ Giao diện người dùng (UI), Logic ứng dụng (Business Logic) và Cơ sở dữ liệu (DBMS/Data) nằm chung trên **một máy tính/máy chủ duy nhất**.
    
- **Đặc tính:**
    
    - Truy cập dữ liệu cực nhanh vì không có độ trễ mạng (_zero network latency_).
        
    - Đơn giản, chi phí triển khai tối thiểu.
        
- **Hạn chế:** Không thể mở rộng (không hỗ trợ scale-out), tính cộng tác bằng 0, chỉ phù hợp cho người dùng đơn lẻ hoặc môi trường phát triển cục bộ (local testing, embedded DB như SQLite, MS Access cá nhân).
    

#### 2.2. Kiến Trúc 2 Tầng (2-Tier Architecture - Client/Server)

- **Cấu trúc:** Mô hình Client - Server trực tiếp:
    
    - **Client Tier:** Chứa giao diện (Presentation) và thường kèm phần lớn logic ứng dụng (Thick/Fat Client).
        
    - **Database Tier:** Máy chủ CSDL tập trung (lưu trữ và xử lý truy vấn dữ liệu).
        
    - Client kết nối thẳng đến DB qua driver mạng (JDBC, ODBC).
        
- **Ứng dụng điển hình:** Các ứng dụng desktop nội bộ (Desktop CRM, phần mềm kế toán mạng LAN kết nối máy chủ SQL nội bộ).
    
- **Hạn chế:**
    
    - Kém an toàn: Client giữ thông tin kết nối và giao tiếp trực tiếp với DB.
        
    - Khó mở rộng: Quá nhiều client mở connection trực tiếp dễ làm nghẽn DB connection pool.
        
    - Khó bảo trì: Khi đổi business logic, phải cập nhật lại ứng dụng trên từng máy client.
        

#### 2.3. Kiến Trúc 3 Tầng / N Tầng (3-Tier / N-Tier Architecture)

- **Cấu trúc:** Tách biệt rõ ràng thành 3 lớp độc lập:
    

$$\text{Client (Presentation Layer)} \iff \text{Application Server (Business Logic)} \iff \text{Database Server (Data Layer)}$$

- **Đặc tính & Lợi thế cốt lõi:**
    
    1. **Bảo mật (Enhanced Security):** DB nằm trong mạng riêng (Private Subnet), chỉ chấp nhận kết nối từ Application Server; Client không bao giờ chạm trực tiếp vào DB.
        
    2. **Khả năng mở rộng (Scalability):** Từng tầng có thể co giãn độc lập (ví dụ: Scale-out cụm Application Servers bằng Load Balancer mà không cần can thiệp DB).
        
    3. **Tách biệt mối quan tâm (Separation of Concerns):** Thuận lợi cho triển khai CI/CD, chuẩn Cloud-Native, và phát triển microservices.
        
- **Ứng dụng:** Chuẩn thiết kế của hầu hết các ứng dụng Web và Mobile hiện đại.

## Distributed architecture

![[Pasted image 20261024132049.png]]
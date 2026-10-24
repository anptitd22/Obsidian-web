---
title: Overview
tags:
  - RDBMS
---

![[introduction.png]]

## 5 Thành Phần Cốt Lõi

| **Thành phần**             | **Vai trò chính**                                | **Cơ chế / Khái niệm kỹ thuật quan trọng**                                                                                                                                                         |
| -------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Client Comm Manager** | Cổng giao tiếp mạng, quản lý kết nối & phiên     | • 2-tier / 3-tier / 4-tier proxy<br><br>• Prefetching / Socket queue buffer                                                                                                                        |
| **2. Process Manager**     | Điều phối tiến trình/luồng & kiểm soát tiếp nhận | • Process-per-worker vs. Thread-per-worker vs. Pool<br><br>• Admission Control 2 tầng (chống thrashing)                                                                                            |
| **3. Query Processor**     | Biên dịch, tối ưu và thực thi SQL                | • Query Parser & Rewrite<br><br>• Cost-based Optimizer & Plan Cache<br><br>• Volcano Iterator model (`get_next()`), SARGs                                                                          |
| **4. Storage Manager**     | Quản lý lưu trữ vật lý & đảm bảo ACID            | • Lock Manager (Strict 2PL, Latches)<br><br>• Log Manager (WAL, STEAL/NOT-FORCE, LSN)<br><br>• Buffer Manager (Dirty bit, Pin count, LRU-2)<br><br>• Access Methods (B+Tree, Heap, SARGs pushdown) |
| **5. Shared Utilities**    | Hỗ trợ hạ tầng dùng chung & bảo trì              | • Catalog Manager (Metadata caching)<br><br>• Context-based Memory Allocator<br><br>• Log-based Replication<br><br>• Online Utilities (Vacuum/Reorg, Stats, Fuzzy dump)                            |

### Client Communications Manager

- **Mô hình kết nối**:
    
    - _Two-tier_: Client kết nối trực tiếp DB (ODBC/JDBC).
        
    - _Three/Four-tier_: Client $\rightarrow$ Web/App Server (Proxy/Pool) $\rightarrow$ DB.
        
- **Cơ chế truyền dữ liệu**:
    
    - **Pull model**: Client liên tục gọi `FETCH` theo lô (batch/tuple).
        
    - **Prefetching**: DB tính toán trước và đẩy dữ liệu vào socket buffer/queue mạng để giảm latency mạng khi client request đợt tiếp theo.

### Process Manager & Admission Control

- **3 mô hình quản lý Worker**:
    
    - **Process-per-worker** _(Postgres, Oracle mặc định)_: Độc lập, an toàn bộ nhớ, nhưng ngốn RAM, chuyển ngữ cảnh (context switch) nặng, phải dùng OS shared memory.
        
    - **Thread-per-worker** _(MySQL, SQL Server)_: Nhẹ, tốn ít RAM, chia sẻ bộ nhớ dễ dàng. Có thể dùng OS threads hoặc tự quản lý User-space lightweight threads.
        
    - **Process/Thread Pool**: Giới hạn số lượng worker cố định, ghép kênh (multiplexing) nhiều kết nối vào một nhóm worker rảnh. Chịu tải cao tốt nhất.
        
- **Admission Control (Kiểm soát tiếp nhận - Chống Thrashing)**:
    
    - _Mục tiêu_: Giúp hệ thống suy thoái nhẹ nhàng (graceful degradation) – duy trì Throughput tối đa thay vì crash khi quá tải.
        
    - _Tầng 1 (Dispatcher)_: Giới hạn trần tổng số kết nối mạng.
        
    - _Tầng 2 (Query Execution)_: Dựa trên ước tính của Optimizer về **Memory Footprint** (sắp xếp/băm), CPU, Disk I/O để quyết định: _Thực thi ngay / Giảm RAM cấp / Tạm hoãn_.

### Relational Query Processor

**Quy trình 4 bước**:

1. **Parsing & Authorization**: Phân tích cú pháp, chuẩn hóa tên (`server.db.schema.table`), tra cứu Catalog để kiểm tra kiểu dữ liệu và quyền truy cập.
    
2. **Query Rewrite**:
    
    - _View Expansion_: Bung mã view vào câu truy vấn gốc.
        
    - _Logic Simplification_: Rút gọn hằng số, biến đổi phủ định (`NOT a > b` $\rightarrow$ `a <= b`), loại bỏ nhánh logic luôn sai.
        
    - _Subquery Flattening_: Làm phẳng subquery lồng nhau để join hiệu quả hơn.
        
3. **Optimizer (Tối ưu hóa)**:
    
    - Dùng thống kê (Histograms) để ước lượng số dòng trả về (Selectivity).
        
    - Thuật toán Dynamic Programming / Top-down tìm cây thực thi có chi phí thấp nhất.
        
    - Lưu vào **Plan Cache** để tái sử dụng kế hoạch biên dịch.
        
4. **Executor (Thực thi)**:
    
    - **Volcano / Iterator model**: Cây toán tử gọi đệ quy hàm `get_next()`, kéo dữ liệu theo luồng dữ liệu (dataflow).
        
    - **SARGs Pushdown**: Đẩy điều kiện lọc xuống tận tầng lưu trữ đĩa để tránh load bản ghi rác lên RAM.
        
    - **Halloween Problem Protection**: Materialize danh sách Record ID trước khi Update/Delete để tránh cập nhật lặp một bản ghi khi nó đổi vị trí trên chỉ mục.

### Transactional Storage Manager (Bộ tứ ACID)

Bốn thành phần đan xen chặt chẽ (tightly coupled):

```
[Query Executor]
       │
       ▼ (SARGs)
[Access Methods] ◄───► [Lock Manager] (Strict 2PL, Latches)
       │
       ▼
[Buffer Manager] (Pin count, Dirty bit, LRU-2)
       │
       ▼ (WAL: Log trước - Data sau)
[Log Manager] (LSN, Fuzzy Checkpoint, Steal/Not-Force)
```

- **Lock Manager**: Quản lý Lock Table & Deadlock Detection. Sử dụng **Strict 2PL** (giữ khóa đến cuối giao dịch) và **Latches** (chốt ngắn hạn bảo vệ cấu trúc RAM).
    
- **Log Manager**: Tuân thủ nguyên tắc **WAL (Write-Ahead Logging)**. Đa số dùng kiến trúc **STEAL / NOT-FORCE** (trang bẩn có thể ghi xuống đĩa trước khi commit, và lúc commit không bắt buộc phải ép flush toàn bộ data pages xuống đĩa ngay, chỉ cần flush log).
    
- **Buffer Manager**: Quản lý Buffer Pool. Mỗi page có:
    
    - `Dirty bit`: Đánh dấu dữ liệu đã bị sửa, cần ghi lại xuống đĩa.
        
    - `Pin count`: Đang có tiến trình đọc/ghi, cấm thay thế trang này khỏi RAM.
        
    - Thuật toán thay thế: Dùng **LRU-2** hoặc Clock sweep để tránh việc quét toàn bảng (Full Table Scan) xóa sạch cache.
        
- **Access Methods**: Cấu trúc Heap files, B+Tree Indexes. Cung cấp API tích hợp SARGs và cơ chế bảo vệ cấu trúc (như Right-link tree, Next-key locking).

### Shared Components and Utilities

- **Catalog Manager**: Lưu metadata hệ thống dưới dạng bảng quan hệ, cache lên RAM bằng mạng lưới đối tượng phi chuẩn hóa để tra cứu tức thì.
    
- **Context-based Memory Allocator**: Cấp phát bộ nhớ theo ngữ cảnh (Memory Context). Khi kết thúc tác vụ (ví dụ: chạy xong Hash Join), giải phóng toàn bộ context cùng lúc $\rightarrow$ Không tốn chi phí gọi OS `malloc/free` liên tục, triệt tiêu Memory Leak.
    
- **Disk Management Subsystem**: Trừu tượng hóa tệp tin OS, xử lý tối ưu trên phần cứng phức tạp (SAN, RAID 1 vs. RAID 5 write penalty).
    
- **Replication Services**: Phương pháp chuẩn hóa là **Log-based Replication** (Log sniffer đọc WAL và replay sang node Standby) – tốn ít tài nguyên, trễ thấp, đảm bảo đúng ngữ nghĩa giao dịch.
    
- **Online Utilities**: Công cụ bảo trì chạy nền 24/7 không khóa bảng:
    
    - _Stats Gathering_: Cập nhật histogram cho Optimizer.
        
    - _Online Reorganization_: Chống phân mảnh bảng/chỉ mục.
        
    - _Fuzzy Backup/Dump_: Sao lưu dữ liệu trực tiếp kết hợp nhật ký WAL.
        
    - _Bulk Load_: Nạp dữ liệu nhanh bằng cách ghi tắt trực tiếp vào data page, bỏ qua parsing/SQL overhead.
        
    - _Resource Governor_: Tự động giới hạn hoặc kill truy vấn ngốn tài nguyên bất thường.

## Luồng Vòng Đời Truy Vấn (End-to-End Query Lifecycle)

_Ví dụ: Nhân viên sân bay ấn nút lấy danh sách hành khách chuyến bay._

1. **Client** gửi lệnh $\rightarrow$ Kết nối qua mạng (JDBC/ODBC, 2-tier / 3-tier / 4-tier).
    
2. **Client Communications Manager**: Xác thực thông tin, lưu trạng thái phiên, chuyển tiếp lệnh SQL vào trong.
    
3. **Process Manager**: Cấp phát luồng tính toán (DBMS worker), kiểm soát tiếp nhận (Admission Control: chạy ngay hay chờ).
    
4. **Relational Query Processor**:
    
    - Parse & Authorize (kiểm tra cú pháp & quyền).
        
    - Rewrite (chuẩn hóa view, logic Boolean).
        
    - Optimize (chọn query plan tối ưu dựa trên thống kê chi phí I/O, CPU, RAM).
        
    - Execute (chạy các toán tử iterator `get_next()`: Index Scan, Filter, Hash Join...).
        
5. **Transactional Storage Manager**:
    
    - Access Methods tương tác với Buffer Pool để đọc trang dữ liệu (Pages) từ đĩa.
        
    - Lock Manager cấp khóa (đảm bảo tính Isolation - 2PL).
        
    - Log Manager ghi WAL nếu có ghi/sửa dữ liệu (đảm bảo Atomicity & Durability).
        
6. **Trả kết quả ("Unwinding the stack")**:
    
    - Dữ liệu đẩy lên các toán tử xử lý $\rightarrow$ đưa vào Client Buffer $\rightarrow$ trả về Client theo mô hình kéo (Pull/Prefetch).
        
    - Dọn dẹp tài nguyên (Transaction $\rightarrow$ Process $\rightarrow$ Communication).



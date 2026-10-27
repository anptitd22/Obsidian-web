## 1. OVERVIEW

- **2 cấp độ lập lịch (Scheduling Levels):**
    
    1. **Between Applications (Giữa các Application):** Do Cluster Manager (YARN, K8s, Standalone, Mesos) quản lý.
        
    2. **Within Application (Trong cùng 1 Application):** Do [[SparkContext]] quản lý (khi gửi nhiều Action/Job từ các luồng khác nhau).

## 2. SCHEDULING ACROSS APPLICATIONS

### A. Phân vùng tĩnh (Static Partitioning)

- **Đặc điểm:** Cấp phát tài nguyên cố định ngay từ đầu, giữ nguyên cho đến khi App kết thúc. Mặc định cho Standalone, YARN và Mesos Coarse-grained.
    
- **Cấu hình theo Cluster Manager:**
    
    - **Standalone:** Chạy mặc định theo thứ tự FIFO.
        
        - `spark.cores.max`: Giới hạn số core tối đa cho 1 App.
            
        - `spark.deploy.defaultCores`: Số core mặc định nếu App không khai báo.
            
        - `spark.executor.memory`: Giới hạn RAM cho mỗi Executor.
            
    - **Mesos (Coarse-grained):** Đặt `spark.mesos.coarse = true`, chỉnh `spark.cores.max` và `spark.executor.memory`.
        
    - **YARN:**
        
        - `--num-executors` (`spark.executor.instances`): Số lượng Executor.
            
        - `--executor-cores` (`spark.executor.cores`): Số CPU core cho mỗi Executor.
            
        - `--executor-memory` (`spark.executor.memory`): Dung lượng RAM cho mỗi Executor.

### B. Chia sẻ CPU động (Dynamic CPU Sharing - Mesos Fine-grained)

- **Cấu hình:** Đặt `spark.mesos.coarse = false`.
    
- **Đặc điểm:** RAM cố định, nhưng CPU core nhàn rỗi có thể cho App khác mượn.
    
- **Đánh đổi:** Độ trễ khó dự đoán do mất thời gian để App lấy lại core khi có Task mới.
    

> **Lưu ý:** Spark **KHÔNG** hỗ trợ chia sẻ bộ nhớ (RAM) giữa các App khác nhau. Muốn chia sẻ dữ liệu, hãy chạy 1 Server App duy nhất để phục vụ nhiều truy vấn trên cùng RDD/DataFrame.

## 3. DYNAMIC RESOURCE ALLOCATION 

Tự động xin thêm Executor khi quá tải và trả lại tài nguyên cho Cluster khi rảnh rỗi.

### A. Điều kiện bật (Configuration & Setup)

- **Bắt buộc:** `spark.dynamicAllocation.enabled = true`
    
- **Và chọn 1 trong các cơ chế bảo toàn dữ liệu Shuffle sau:**
    
    1. Bật External Shuffle Service: `spark.shuffle.service.enabled = true`
        
    2. Bật Shuffle Tracking: `spark.dynamicAllocation.shuffleTracking.enabled = true`
        
    3. Bật Decommission: `spark.decommission.enabled = true` **VÀ** `spark.storage.decommission.shuffleBlocks.enabled = true`
        
    4. Cấu hình `spark.shuffle.sort.io.plugin.class` sử dụng Custom ShuffleDataIO hỗ trợ Reliable Storage.
        

### B. Chính sách xin thêm Executor (Request Policy)

- **Cơ chế:** Kích hoạt theo từng vòng (Round) khi có Task chờ xếp lịch (Pending Backlog).
    
- **Thời gian chờ:**
    
    - Vòng 1: Sau khi chờ `spark.dynamicAllocation.schedulerBacklogTimeout` (giây).
        
    - Các vòng tiếp theo: Sau mỗi `spark.dynamicAllocation.sustainedSchedulerBacklogTimeout` (giây).
        
- **Tốc độ tăng:** Theo cấp số nhân ($1 \to 2 \to 4 \to 8 \dots$) để vừa cẩn trọng lúc đầu, vừa tăng tốc nhanh khi cần thiết.
    

### C. Chính sách thu hồi Executor (Remove Policy)

- **Cơ chế:** Thu hồi khi Executor không làm việc trong khoảng thời gian vượt quá `spark.dynamicAllocation.executorIdleTimeout` (giây).

### D. Bảo toàn trạng thái (Graceful Decommission)

- **Vấn đề:** Khi bị thu hồi, Executor sẽ làm mất dữ liệu Shuffle & Cache mà nó đang lưu giữ.
    
- **Giải pháp:**
    
    - **Shuffle Data:** Sử dụng **External Shuffle Service** — tiến trình chạy dài hạn mỗi node của cluster, độc lập với các Spark application và các executors của chúng. 

	    - Nếu dịch vụ được bật, các Spark executors sẽ lấy các tệp shuffle từ dịch vụ thay vì lấy từ nhau.
        
    - **Cache Data:** Mặc định Executor chứa Cache **sẽ không bị thu hồi**.
        
        - Chỉnh thời gian chờ thu hồi Executor có Cache qua `spark.dynamicAllocation.cachedExecutorIdleTimeout`.
            
        - Bật `spark.shuffle.service.fetch.rdd.enabled = true` để hỗ trợ lấy khối RDD lưu trên đĩa qua External Shuffle Service.

## 4. SCHEDULING WITHIN AN APPLICATION

Khi một App chạy nhiều luồng (multithreading) gửi các Action đồng thời, Spark hỗ trợ 2 chế độ lập lịch:

### A. FIFO (First In, First Out) - Mặc định

- Job gửi trước chạy trước.
    
- Job ở đầu sẽ chiếm trọn tài nguyên của Cluster. Các Job sau phải đợi cho đến khi Job đầu giải phóng bớt tài nguyên.
    

### B. FAIR Scheduler (Phân chia công bằng)

- Phân chia tài nguyên theo kiểu Round-Robin, giúp các Job ngắn không bị nghẽn bởi Job dài.
    
- **Cách bật:**

```scala
val conf = new SparkConf().set("spark.scheduler.mode", "FAIR")
val sc = new SparkContext(conf)
```

## 5. FAIR SCHEDULER POOLS

Giúp nhóm các Job lại thành các Pool (ví dụ: Pool ưu tiên cao, Pool theo User).

### A. Gán Job vào Pool trong Code

```scala
// Gán luồng hiện tại vào pool1 sc.setLocalProperty("spark.scheduler.pool", "pool1") 

// Xóa thiết lập pool (quay về default) sc.setLocalProperty("spark.scheduler.pool", null)
```

### B. Cấu hình thuộc tính của Pool (XML)

Tạo file `fairscheduler.xml` và khai báo đường dẫn: `conf.set("spark.scheduler.allocation.file", "file:///path/to/file")`.

**3 Thuộc tính chính của mỗi Pool:**

1. **`schedulingMode`:** `FIFO` hoặc `FAIR` (lập lịch cho các Job _bên trong_ Pool đó).
    
2. **`weight`:** Trọng số chia tài nguyên so với Pool khác (Mặc định = 1. Ví dụ Weight = 2 sẽ nhận gấp đôi tài nguyên).
    
3. **`minShare`:** Số CPU core tối thiểu được cam kết cấp cho Pool trước khi chia phần dư theo `weight` (Mặc định = 0).

```xml
<?xml version="1.0"?>
<allocations>
  <pool name="production">
    <schedulingMode>FAIR</schedulingMode>
    <weight>1</weight>
    <minShare>2</minShare>
  </pool>
  <pool name="test">
    <schedulingMode>FIFO</schedulingMode>
    <weight>2</weight>
    <minShare>3</minShare>
  </pool>
</allocations>
```

## 6. TRƯỜNG HỢP ĐẶC BIỆT

**JDBC/Thrift Server:** Đặt Pool cho từng phiên truy vấn bằng lệnh SQL:

```sql
SET spark.sql.thriftserver.scheduler.pool=accounting;
```

**PySpark:** Do sự bất đồng bộ giữa luồng Python (PVM) và Java (JVM), việc thiết lập Pool qua `sc.setJobGroup` trên các luồng PVM riêng biệt hoặc hủy Job qua `sc.cancelJobGroup` **không được hỗ trợ ổn định mặc định**.
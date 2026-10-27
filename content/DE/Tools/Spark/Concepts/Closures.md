### 1. Closure là gì?

- **Định nghĩa:** Là tập hợp các biến (variables) và phương thức (methods) mà Spark cần để thực hiện các phép tính trên RDD.
    
- **Cơ chế hoạt động:**
    
    1. Spark chia nhỏ code thành các **task** để chạy trên các **Executor**.
        
    2. Spark **serialize** (tuần tự hóa) các biến/hàm cần thiết (closure) từ **Driver** và gửi một bản sao độc lập đến từng Executor.

### 2. Vấn đề thường gặp (The Trap)

- **Tình huống:** Định nghĩa một biến toàn cục ở Driver (ví dụ: `counter = 0`) rồi cập nhật biến đó bên trong một hàm chạy trên Executor (như `rdd.foreach(...)`).

```
var counter = 0
var rdd = sc.parallelize(data)
rdd.foreach(x => counter += x)
println("Counter value: " + counter)
```

- **Kết quả lỗi:**
    
    - **Trên Distributed Cluster:** Mỗi Executor chỉ sửa đổi **bản sao** của biến đó trên JVM riêng của nó. Biến `counter` trên Driver hoàn toàn không đổi (vẫn bằng `0`).
        
    - **Trên Local Mode (1 máy):** Code có thể vô tình chạy đúng vì Driver và Executor chung JVM, nhưng đây là kết quả **ngẫu nhiên (by accident)** và không được Spark bảo đảm.

### 3. Quy tắc cốt lõi (Core Rule)

> **Không bao giờ dùng closure để thay đổi trạng thái toàn cục (mutate global state).** Spark không bảo đảm hành vi của các biến bên ngoài bị biến đổi bên trong closure.

### 4. Giải pháp thay thế (What to do instead)

- **Tổng hợp phân tán (Distributed Aggregation):** Sử dụng [[Accumulators]] khi cần cập nhật biến từ nhiều node khác nhau.
    
- **Đếm phần tử / Thống kê:** Ưu tiên dùng các action có sẵn của Spark như `.count()`, `.reduce()`, hoặc các hàm của DataFrame API.

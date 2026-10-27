## 1. OVERVIEW

- **`bin/spark-submit`:** Tập lệnh (script) duy nhất dùng để khởi chạy Spark application trên mọi Cluster Manager (Standalone, YARN, K8s, Mesos).
    
- **Ưu điểm:** Cung cấp một giao diện chung (interface thống nhất), không cần thay đổi mã nguồn khi chuyển đổi giữa các loại cluster.

## 2. BUNDLING DEPENDENCIES

### A. Dành cho Java / Scala

- Đóng gói mã nguồn và thư viện phụ thuộc thành file **Uber JAR** (hoặc Fat JAR) bằng plugin của `sbt` hoặc `Maven`.
    
- **Quy tắc quan trọng:** Phải đặt phạm vi (scope) của **Spark** và **Hadoop** là `provided` (chỉ dùng để biên dịch, không đóng gói vào JAR vì Cluster Manager đã cung cấp sẵn ở runtime).
    

### B. Dành cho Python (PySpark)

- Phân phối file nguồn bằng đối số `--py-files` (nhận các định dạng `.py`, `.zip`, `.egg`).
    
- Nếu dự án có nhiều file Python, nên nén lại thành file `.zip` hoặc `.egg`.

## 3. CÚ PHÁP VÀ TÙY CHỌN CỦA SPARK-SUBMIT

### A. Cú pháp chuẩn


```bash
./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  <application-jar-hoặc-python-file> \
  [application-arguments]
```

### B. Các tham số phổ biến

- **`--class`:** Entry point của ứng dụng (chứa hàm `main`, áp dụng cho Java/Scala). _VD: `org.apache.spark.examples.SparkPi`_
    
- **`--master`:** URL của Cluster Master.
    
- **`--deploy-mode`:** Chế độ triển khai Driver, gồm 2 loại:
    
    - **`client` (Mặc định):** Driver chạy ngay tại máy thực thi lệnh `spark-submit` (bên ngoài cluster). Phù hợp khi gửi lệnh từ Gateway Node chung mạng vật lý với Worker, hoặc chạy REPL (Spark Shell).
        
    - **`cluster`:** Driver được đẩy vào chạy bên trong một Worker node thuộc cụm. Phù hợp khi submit lệnh từ xa (VD: laptop cá nhân) để giảm độ trễ mạng giữa Driver và Executors.
        
- **`--conf`:** Cấu hình tùy chỉnh dạng `key=value`. Nếu giá trị có dấu cách, bọc trong dấu `""`.
    
- **`--supervise`:** (Chỉ dùng cho Standalone & Mesos ở `cluster` mode) Tự động khởi động lại Driver nếu gặp lỗi (mã thoát khác 0).
    

> **Lưu ý:** Standalone mode **không hỗ trợ** `--deploy-mode cluster` cho các ứng dụng Python.

## 4. BẢNG TRA CỨU MASTER URL (`--master`)

|**Master URL**|**Ý nghĩa / Mô tả**|
|---|---|
|`local`|Chạy local với đúng 1 thread (không song song).|
|`local[K]`|Chạy local với `K` worker threads (nên đặt bằng số CPU cores của máy).|
|`local[K,F]`|Chạy local với `K` threads và `F` lần thử lại tối đa (`maxFailures`).|
|`local[*]`|Chạy local với số threads bằng đúng số logical CPU cores của máy.|
|`local[*,F]`|Chạy local với tất cả logical cores và `F` lần thử lại tối đa.|
|`local-cluster[N,C,M]`|Giả lập cụm phân tán trong 1 JVM (dùng cho Unit Test): `N` workers, `C` cores/worker, `M` MiB RAM/worker.|
|`spark://HOST:PORT`|Kết nối Spark Standalone Master (Port mặc định: `7077`).|
|`spark://HOST1:PORT1,HOST2:PORT2`|Kết nối Spark Standalone Cluster cấu hình High Availability (HA) qua ZooKeeper.|
|`mesos://HOST:PORT`|Kết nối Mesos Cluster (Port mặc định: `5050`). Dùng `mesos://zk://...` nếu dùng ZooKeeper.|
|`yarn`|Kết nối YARN Cluster (tự động tìm cấu hình qua biến môi trường `HADOOP_CONF_DIR` hoặc `YARN_CONF_DIR`).|
|`k8s://HOST:PORT`|Kết nối Kubernetes API Server (mặc định dùng TLS/HTTPS). Sử dụng `k8s://http://...` để dùng HTTP.|
## 5. CONFIGURATION PRIORITY

Spark áp dụng cấu hình theo thứ tự ưu tiên từ cao xuống thấp như sau:

1. **Mức cao nhất:** Đặt trực tiếp bằng mã nguồn thông qua đối tượng `SparkConf`.
    
2. **Mức 2:** Các cờ/tham số truyền trực tiếp trong lệnh `spark-submit` (như `--master`, `--executor-memory`).
    
3. **Mức thấp nhất:** Các giá trị khai báo trong file cấu hình mặc định `conf/spark-defaults.conf`.
    

> **Mẹo:**
> 
> - Nếu đã đặt `spark.master` trong file `spark-defaults.conf`, bạn có thể bỏ qua cờ `--master` khi gõ lệnh `spark-submit`.
>     
> - Thêm cờ `--verbose` vào lệnh `spark-submit` để debug và xem chi tiết nguồn gốc của các thông số cấu hình.

## 6. QUICK EXAMPLES

**Chạy Local (8 Cores):**

```bash
./bin/spark-submit --class org.apache.spark.examples.SparkPi --master local[8] /path/to/examples.jar 100
```

**Chạy trên YARN (Cluster Mode):**

```bash
export HADOOP_CONF_DIR=/path/to/hadoop/conf
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master yarn \
  --deploy-mode cluster \
  --executor-memory 20G \
  --num-executors 50 \
  /path/to/examples.jar 1000
```

**Chạy PySpark trên Standalone Cluster:**

```bash
./bin/spark-submit \
  --master spark://207.184.161.138:7077 \
  examples/src/main/python/pi.py 1000
```

**Chạy trên Kubernetes (Cluster Mode):**

```bash
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://xx.yy.zz.ww:443 \
  --deploy-mode cluster \
  --executor-memory 20G \
  --num-executors 50 \
  http://path/to/examples.jar 1000
```
## 1. Cơ chế hoạt động (How It Works)

![[Pasted image 20261029234828.png]]

- **Trình lập lịch tích hợp (Native Scheduler)**: Spark tích hợp sẵn trình lập lịch K8s native, cho phép gửi ứng dụng trực tiếp tới cụm K8s thông qua lệnh `spark-submit`.

- **Luồng thực thi**:
    1. `spark-submit` tạo một **Driver Pod** trên K8s.
    2. Driver Pod giao tiếp với K8s API server để tạo các **Executor Pods** và thực thi mã nguồn.
    3. Khi hoàn thành, các Executor Pods bị kết thúc và giải phóng tài nguyên. Driver Pod chuyển sang trạng thái **"Completed"** (giữ lại log, không tốn CPU/RAM) cho đến khi bị dọn dẹp.

## 2. Yêu cầu tiên quyết (Prerequisites)

- **Cụm Kubernetes**: Phiên bản **>= 1.24**, đã cấu hình kết nối qua `kubectl`. _(Chạy thử nghiệm local với_ _minikube_ _khuyến nghị tối thiểu 3 CPUs và 4GB RAM)_.

- **Quyền hạn RBAC**: Service Account dùng cho Driver Pod phải có quyền khởi tạo, liệt kê, chỉnh sửa và xóa `pods`, `services`, `configmaps`.

- **Mạng**: K8s DNS phải được bật trong cụm.

## 3. Chế độ triển khai (Deploy Modes)

### A. Cluster Mode (Driver chạy trong Pod)

- **Đặc điểm**: Driver chạy bên trong một Pod trên cụm K8s.

- **Cú pháp Master**: `--master k8s://https://<k8s-apiserver-host>:<port>` _(Luôn phải chỉ định cổng, kể cả cổng HTTPS 443)_.

- **Tên ứng dụng**: Tên truyền vào (`--name`) được dùng để đặt tên cho các K8s Pods (phải viết chữ thường, số, dấu `-`, `.`).

### B. Client Mode (Driver chạy ngoài Pod / Máy cục bộ)

- **Đặc điểm**: Driver chạy trực tiếp trên máy gửi job (hoặc một Pod bên ngoài).

- **Cấu hình kết nối Mạng**: Executor phải kết nối được ngược về Driver qua `spark.driver.host` và `spark.driver.port`. Nếu Driver nằm trong 1 Pod khác, nên dùng **Headless Service**.

- **Thu gom rác Executor (GC)**: Nên đặt `spark.kubernetes.driver.pod.name` để Spark gắn `OwnerReference`, tự động xóa tất cả Executor Pods khi Driver Pod bị xóa.

## 4. Quản lý Container Images & Bảo mật (Docker & Security)

- **Công cụ đóng gói Image**: Spark cung cấp script `./bin/docker-image-tool.sh` để build và push Docker images (hỗ trợ JVM, PySpark, SparkR).

- **Danh tính Người dùng (User Identity)**: Mặc định chạy dưới dạng unprivileged UID **185**.

- **Mã hóa & Xác thực**: Không bật mặc định. Cần giới hạn quyền mount `hostPath` vì nguy cơ lỗ hổng bảo mật đĩa hệ thống.

## 5. Quản lý Cấu hình & Tài nguyên nâng cao

### A. Pod Template (Mẫu thiết kế Pod)

- Cho phép định nghĩa cấu hình Pod bằng tệp YAML chuẩn của K8s qua thuộc tính `spark.kubernetes.driver.podTemplateFile` và `spark.kubernetes.executor.podTemplateFile`.

### B. Quản lý Bộ nhớ & Tệp trung gian (Volumes & Local Storage)

- **Các loại Volume hỗ trợ**: `hostPath`, `emptyDir`, `nfs`, `persistentVolumeClaim` (PVC).

- **Lưu trữ Shuffle (Local Storage)**:
    - Mặc định dùng `emptyDir` (xóa khi Pod dừng).
    - Có thể bật `spark.kubernetes.local.dirs.tmpfs = true` để dùng **RAM (tmpfs)** giúp tăng tốc I/O.
    - Hỗ trợ gán **On-Demand PVC** cho Executor khi thực hiện các phép Shuffle/Sort dung lượng lớn hoặc sử dụng cơ chế Dynamic Allocation.

### C. Trình lập lịch tùy chỉnh (Custom Schedulers)

- Hỗ trợ tích hợp các Scheduler nâng cao cho môi trường Batch/Big Data:
    - **Volcano**: Hỗ trợ Queue Scheduling, PodGroup, và Resource Reservation.
    - **Apache YuniKorn**: Hỗ trợ công bằng tài nguyên (Resource Fairness), giới hạn Queue Min/Max.

## 6. Giám sát, Gỡ lỗi & Quản lý Job (Monitoring & Debugging)

- **Xem Log Driver**: `kubectl logs -f <driver-pod-name>`.

- **Truy cập Spark UI (Cổng 4040)**: Dùng lệnh `kubectl port-forward <driver-pod-name> 4040:4040`.

- **Hủy Job (Kill Application)**:
    - Lệnh: `spark-submit --kill <namespace>:<driver-pod-name> --master k8s://...`.
    - Xóa Driver Pod trên K8s cũng đồng thời dọn dẹp toàn bộ ứng dụng và các Executor Pods liên quan.
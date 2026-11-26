### 1. Helm là gì?

- **Định nghĩa:** Là trình quản lý gói (package manager) cho Kubernetes (tương tự `apt`, `yum` trên Linux hay `Homebrew` trên macOS).
    
- **Mục đích:** Giúp định nghĩa, cài đặt và nâng cấp ứng dụng trên Kubernetes từ đơn giản đến phức tạp.
    
- **Giải quyết vấn đề:** Thay vì quản lý thủ công hàng loạt file manifest rời rạc (Deployment, Service, ConfigMap...) dễ sai sót qua nhiều môi trường, Helm đóng gói tất cả thành một đơn vị thống nhất để dễ dàng phiên bản hóa (versioning), chia sẻ và rollback.
    

### 2. Helm làm được gì?

- **Cài đặt & Quản lý:** Triển khai nhanh các ứng dụng có sẵn (database, monitoring, ingress...) mà không cần tự viết manifest từ đầu.
    
- **Đóng gói & Chia sẻ:** Gom tài nguyên K8s thành gói để tái sử dụng nội bộ hoặc chia sẻ cho cộng đồng.
    
- **Cấu hình đa môi trường:** Dùng chung 1 gói nhưng truyền vào các bộ giá trị (values) khác nhau cho từng môi trường (`dev`, `staging`, `prod`) mà không cần nhân bản file cấu hình.
    
- **Upgrade & Rollback an toàn:** Nâng cấp phiên bản mới hoặc quay lui (rollback) về bản cũ ngay lập tức nếu có sự cố.
    
- **Quản lý Dependencies:** Khai báo các thành phần phụ thuộc để Helm tự động cài đặt đồng bộ cùng lúc.
    

### 3. Đối tượng sử dụng Helm

_(Lưu ý: Helm chỉ tập trung vào **ứng dụng** chạy trên cluster, không quản lý hạ tầng của cluster như control plane hay nodes)._

|**Vai trò**|**Nhiệm vụ chính**|
|---|---|
|**Application Operator**|Triển khai và vận hành ứng dụng bên trong K8s cluster (VD: chạy WordPress + MySQL).|
|**Application Distributor**|Đóng gói ứng dụng thành các gói chuẩn để người khác sử dụng.|
|**Application Developer**|Lập trình ứng dụng (ít quan tâm đến hạ tầng nơi ứng dụng chạy).|
|**Supporting Tool Developer**|Phát triển công cụ bổ trợ cho Helm (VD: plugin, linter).|

### 4. Ba thành phần cốt lõi (Key Components)

1. **Chart (Gói):**
    
    - Là một gói Helm chứa toàn bộ định nghĩa tài nguyên K8s để chạy một ứng dụng/dịch vụ.
        
2. **Repository (Kho lưu trữ):**
    
    - Nơi lưu trữ và chia sẻ các Chart (có thể tìm kiếm các chart công khai trên **Artifact Hub**).
        
3. **Release (Bản phát hành):**
    
    - Là một thực thể (instance) của Chart đang chạy thực tế trên K8s cluster.
        
    - **Cơ chế:** `Release = Chart + Configuration (thường từ file values.yaml)`.
        
    - Một Chart có thể cài đặt nhiều lần trên cùng 1 cluster, mỗi lần tạo ra một Release độc lập với tên riêng (VD: cài 2 database MySQL tách biệt).
        

### 5. Kiến trúc (Architecture)

- **Ngôn ngữ & Giao tiếp:** Viết bằng ngôn ngữ **Go**, giao tiếp trực tiếp với Kubernetes API server qua giao thức **REST/JSON**. Cấu hình viết bằng **YAML**.
    
- **Cấu trúc 2 phần:**
    
    - **Helm Client (CLI):** Chạy trên máy local của người dùng, dùng để phát triển chart, quản lý repo và gửi lệnh thực thi.
        
    - **Helm Library:** Chứa logic cốt lõi để tạo release, cài đặt, nâng cấp hoặc gỡ bỏ chart bằng cách tương tác với K8s API server.
        
- **Lưu trữ dữ liệu:** Helm không cần database riêng mà lưu thẳng thông tin trạng thái của các Release vào **Kubernetes Secrets** bên trong cluster.
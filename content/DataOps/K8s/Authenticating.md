### 1. Tổng quan: Vị trí của Authentication trong luồng Request

Mọi request gửi tới **Kubernetes API Server** (từ `kubectl`, dashboard hoặc từ chính các Pod bên trong cluster) đều phải đi qua 3 cửa ải bảo mật theo thứ tự nghiêm ngặt:

1. **Authentication (Xác thực):** _Bạn là ai?_ (Xác minh danh tính người/dịch vụ gửi request). Nếu thất bại $\rightarrow$ trả về lỗi `401 Unauthorized`.
    
2. **Authorization (Phân quyền):** _Bạn được làm gì?_ (Kiểm tra quyền qua RBAC, ABAC, Webhook...). Nếu thất bại $\rightarrow$ trả về lỗi `403 Forbidden`.
    
3. **Admission Control (Kiểm soát đầu vào):** _Request của bạn có hợp lệ với quy chuẩn cluster không?_ (Kiểm tra quota, security policy, mutating/validating webhooks trước khi lưu vào etcd).
    

### 2. Hai nhóm đối tượng (Users) trong Kubernetes

Kubernetes **không** quản lý tài khoản người dùng bình thường (Human Users) trong database nội bộ (`etcd`), mà chia đối tượng xác thực thành 2 loại:

|**Đặc điểm**|**Normal Users (Người dùng thực / Admin / Dev)**|**Service Accounts (Tài khoản dịch vụ)**|
|---|---|---|
|**Mục đích**|Dành cho con người hoặc hệ thống bên ngoài tương tác với cluster.|Dành cho các **Pod / ứng dụng chạy bên trong** cluster giao tiếp với API Server.|
|**Quản lý bởi**|**Bên ngoài K8s** (Active Directory, Google/GitHub OIDC, file chứng chỉ, hoặc hệ thống xác thực ngoài).|**Bên trong K8s** (là một API resource thực thụ: `kind: ServiceAccount`, gắn theo Namespace).|
|**Cách tạo**|Không có lệnh `kubectl create user`. K8s chỉ tin tưởng thông tin danh tính sau khi xác thực thành công.|Tạo bằng `kubectl create serviceaccount <tên>` (mỗi namespace có sẵn 1 SA `default`).|

### 3. Các phương thức Authentication phổ biến

API Server có thể kích hoạt nhiều module xác thực cùng lúc; module nào xác thực thành công trước sẽ được chấp nhận:

- **X.509 Client Certificates (Chứng chỉ số):**
    
    - Phổ biến nhất cho Admin hoặc các thành phần hệ thống (kubelet, controller-manager).
        
    - K8s đọc trường `CN` (Common Name) làm **Username** và trường `O` (Organization) làm **Group**.
        
    - Thường được nhúng thẳng vào file `~/.kube/config`.
        
- **Service Account Tokens (JWT):**
    
    - Dùng cho các Pod gọi ngược lại API Server (VD: Prometheus, ArgoCD, hoặc app tự đọc ConfigMap).
        
    - K8s tự động mount một Projected Volume chứa JWT token ngắn hạn (có thời hạn hết hạn và tự xoay vòng) vào đường dẫn `/var/run/secrets/kubernetes.io/serviceaccount/token` bên trong Pod.
        
- **OpenID Connect (OIDC) Tokens:**
    
    - Tiêu chuẩn cho doanh nghiệp (SSO). Kết nối K8s với các nhà cung cấp danh tính như **Keycloak, Okta, Azure AD (Entra ID), Google, GitHub**.
        
    - User đăng nhập qua trình duyệt để lấy `id_token` (JWT) và gửi kèm trong header của `kubectl`.
        
- **Static Token / Password File (Không khuyến nghị cho Production):**
    
    - Lưu danh sách token hoặc user/password trong file CSV trên master node và truyền cờ `--token-auth-file` cho API Server. Rất kém an toàn vì khó thu hồi (revoke) mà không khởi động lại API Server.
        
- **Authenticating Proxy / Webhook Token Authentication:**
    
    - Ủy quyền hoàn toàn việc xác thực cho một dịch vụ bên ngoài (như AWS IAM Authenticator trên EKS, hoặc proxy như Pinniped/Teleport). API Server gửi token sang webhook để hỏi xem token đó thuộc về user nào.
        

### 4. Cách hoạt động thực tế với `kubeconfig`

Khi bạn gõ lệnh `kubectl get pods`, `kubectl` sẽ đọc file `~/.kube/config` (gồm 3 phần chính: `clusters`, `users`, `contexts`) để lấy thông tin định danh (client certificate, OIDC token hoặc lệnh exec lấy token của cloud provider) và đính kèm vào HTTP Request gửi lên API Server.

### 5. Điểm cần lưu ý khi vận hành (Best Practices)

- **Không dùng chung 1 Admin Certificate cho cả team:** Khi một nhân sự nghỉ việc, việc thu hồi (revoke) chứng chỉ X.509 trong K8s rất phức tạp (do K8s không có CRL tích hợp sẵn). Hãy dùng **OIDC** cho người dùng thực.
    
- **Không dùng ServiceAccount `default` cho ứng dụng:** Luôn tạo ServiceAccount riêng cho từng ứng dụng với quyền tối thiểu (Least Privilege), và tắt tự động mount token (`automountServiceAccountToken: false`) nếu Pod không cần gọi K8s API.
    
- **Authentication thành công không đồng nghĩa có quyền:** Một user mới xác thực thành công nhưng chưa được gán Role/ClusterRole (Authorization) thì mặc định sẽ không làm được bất cứ thao tác nào trên cluster.
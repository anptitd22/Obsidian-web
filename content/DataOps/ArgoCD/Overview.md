**ArgoCD** là một công cụ **Continuous Delivery (CD)** mã nguồn mở được thiết kế chuyên biệt cho **Kubernetes**, hoạt động theo triết lý **GitOps**.

Nếu như GitHub Actions hay GitLab CI lo phần **CI** (kiểm thử code, build Docker image), thì ArgoCD đảm nhận phần **CD** (lấy cấu hình mới nhất và triển khai lên cụm Kubernetes). Điểm đặc biệt của ArgoCD là nó sử dụng **Git làm "nguồn chân lý duy nhất" (Single Source of Truth)** cho toàn bộ trạng thái hạ tầng và ứng dụng của bạn.

![[Pasted image 20261116002423.png]]

## 1. Cơ chế hoạt động cốt lõi: Mô hình Pull (Kéo)

Trong quy trình CI/CD truyền thống (mô hình **Push**), công cụ CI (như Jenkins, GitLab CI) sau khi build xong sẽ chạy lệnh `kubectl apply` hoặc `helm upgrade` để đẩy cấu hình vào Kubernetes. Cách này buộc bạn phải cấp quyền truy cập trực tiếp (kubeconfig/admin token) của cụm Kubernetes cho công cụ CI bên ngoài, tiềm ẩn rủi ro bảo mật.

ArgoCD đảo ngược quy trình này bằng mô hình **Pull**:

1. **Cài đặt bên trong Cluster:** ArgoCD chạy như một tập hợp các Pod nằm ngay bên trong cụm Kubernetes của bạn.
    
2. **Theo dõi liên tục:** ArgoCD liên tục giám sát kho lưu trữ Git (nơi chứa các file cấu hình Kubernetes YAML, Helm charts, hoặc Kustomize) và đối chiếu với trạng thái thực tế đang chạy trong Kubernetes.
    
3. **Phát hiện sai lệch (OutOfSync):** Khi bạn sửa file cấu hình trên Git (ví dụ: đổi tag của Docker image từ `v1.0` sang `v1.1` hoặc tăng số lượng replica từ `2` lên `5`), ArgoCD nhận thấy trạng thái trên Git khác với trạng thái trong Cluster.
    
4. **Tự động đồng bộ (Sync):** ArgoCD tự động kéo cấu hình mới về và cập nhật cụm Kubernetes cho khớp với Git.
    

## 2. Tại sao ArgoCD lại phổ biến trong hệ sinh thái Kubernetes?

- **Tự động phục hồi cấu hình (Self-Healing):** Nếu ai đó vô tình (hoặc cố ý) dùng lệnh `kubectl edit` sửa tay trực tiếp trên cụm Kubernetes làm sai lệch với file cấu hình trên Git, ArgoCD sẽ lập tức phát hiện trạng thái `OutOfSync` và tự động ghi đè lại đúng theo định nghĩa trên Git.
    
- **Rollback (Quay lui phiên bản) cực nhanh:** Vì mọi cấu hình đều nằm trên Git, nếu phiên bản mới triển khai bị lỗi, bạn chỉ cần thực hiện lệnh `git revert` trên Git (hoặc bấm nút Rollback trên giao diện Web của ArgoCD). Cụm Kubernetes sẽ lập tức quay về trạng thái ổn định trước đó.
    
- **Giao diện trực quan (Web UI):** ArgoCD cung cấp giao diện web rất trực quan giúp bạn nhìn thấy toàn bộ cây quan hệ của một ứng dụng trên Kubernetes (từ Application $\rightarrow$ Deployment $\rightarrow$ ReplicaSet $\rightarrow$ Pod, Service, ConfigMap, Secret) cùng trạng thái sức khỏe (`Healthy` / `Degraded`) và log của từng Pod theo thời gian thực.
    
- **Quản lý đa cụm (Multi-Cluster):** Một hệ thống ArgoCD duy nhất có thể quản lý và triển khai ứng dụng đồng thời lên nhiều cụm Kubernetes khác nhau (Dev, Staging, Production, hoặc nhiều cloud provider khác nhau).
    

## 3. Phân biệt vai trò: Công cụ CI vs. ArgoCD

Để dễ hình dung cách chúng phối hợp với nhau trong một hệ thống hoàn chỉnh:

| **Giai đoạn**              | **Công cụ đảm nhiệm (Ví dụ: GitLab CI / GitHub Actions)**                      | **Công cụ đảm nhiệm (ArgoCD)**                                       |
| -------------------------- | ------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| **1. Xử lý mã nguồn**      | Chạy test, lint code, build Docker image mới.                                  | Không can thiệp vào source code ứng dụng.                            |
| **2. Lưu trữ Artifact**    | Đẩy Docker image lên Registry (Docker Hub, Harbor).                            | Không build hay lưu trữ image.                                       |
| **3. Cập nhật cấu hình**   | Tự động commit & push tag image mới vào **GitOps Repo** (chứa file YAML/Helm). | Liên tục lắng nghe sự thay đổi tại **GitOps Repo**.                  |
| **4. Triển khai (Deploy)** | Không cần kết nối trực tiếp vào cụm Kubernetes.                                | Kéo file YAML/Helm mới từ GitOps Repo và áp dụng lên **Kubernetes**. |

_Lưu ý thực tế:_ Trong các hệ thống chuẩn mực, người ta thường tách làm **2 kho Git riêng biệt**: một **App Repo** (chứa mã nguồn ứng dụng Python, Go, Java...) và một **Manifest/GitOps Repo** (chỉ chứa file Helm values hoặc Kubernetes YAML). Khi App Repo build xong image mới, nó sẽ cập nhật tag image sang GitOps Repo để kích hoạt ArgoCD triển khai.
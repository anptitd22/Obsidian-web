Việc kết hợp **ArgoCD + Helm** là một trong những kiến trúc chuẩn mực và phổ biến nhất (Best Practice) khi vận hành Kubernetes ở quy mô doanh nghiệp. Hai công cụ này không thay thế nhau mà **bổ trợ cho nhau**: **Helm lo phần đóng gói và tham số hóa cấu hình**, còn **ArgoCD lo phần đồng bộ và quản lý vòng đời triển khai**.

![[Pasted image 20261116011947.png]]

## 1. Tại sao dùng ArgoCD rồi vẫn cần Helm?

Nếu chỉ dùng Kubernetes YAML thuần (plain YAML), khi công ty bạn có nhiều môi trường (Dev, Staging, Production) hoặc nhiều dịch vụ giống nhau, bạn sẽ phải copy-paste hàng chục file YAML dài hàng trăm dòng và sửa tay từng thông số (như số lượng Pod, dung lượng RAM, tên miền, biến môi trường).

**Helm** giải quyết bài toán này bằng cách đóng vai trò như một **trình quản lý gói (Package Manager)** và **bộ tạo khuôn mẫu (Template Engine)** cho Kubernetes:

- **Tái sử dụng cấu hình (Templating):** Bạn chỉ cần viết một bộ khung YAML chuẩn (`templates/`) có chứa các biến giữ chỗ (ví dụ: `{{ .Values.replicaCount }}` hoặc `{{ .Values.image.tag }}`).
    
- **Tách biệt cấu hình theo môi trường:** Mỗi môi trường chỉ cần một file giá trị gọn nhẹ riêng biệt: `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml`.
    
- **Cài đặt nhanh công cụ bên thứ ba:** Các hệ thống lớn (như Airflow, Spark, Kafka, Prometheus, Grafana) đều phát hành sẵn Helm Chart chính thức. Bạn chỉ cần lấy về và truyền file `values.yaml` của công ty vào là chạy được ngay, không cần tự viết YAML từ đầu.
    

## 2. ArgoCD và Helm phối hợp với nhau như thế nào?

Trong thực tế, ArgoCD có khả năng **đọc và hiểu trực tiếp Helm Chart** mà không cần bạn phải cài đặt Helm CLI trên từng máy hay chạy lệnh `helm install` thủ công. Quy trình diễn ra qua 4 bước:

1. **Khai báo Application trên ArgoCD:** Bạn tạo một resource `Application` của ArgoCD, trỏ nguồn tới thư mục chứa Helm Chart trên Git (hoặc Helm Repository) và chỉ định file `values-prod.yaml` tương ứng.
    
2. **Render (Kết xuất) cấu hình:** Thành phần `argocd-repo-server` bên trong ArgoCD sẽ tự động chạy lệnh `helm template` ngầm bên dưới. Nó lấy bộ khung `templates/` kết hợp với dữ liệu trong `values-prod.yaml` để sinh ra các file Kubernetes YAML thuần túy hoàn chỉnh.
    
3. **So sánh trạng thái (Diff):** ArgoCD lấy bộ YAML vừa render xong để so sánh với những gì đang chạy thực tế trên cụm Kubernetes.
    
4. **Áp dụng (Sync):** Nếu có thay đổi (ví dụ CI vừa cập nhật `image.tag` mới trong `values-prod.yaml`), ArgoCD áp dụng bộ YAML đó lên Kubernetes thông qua cơ chế `kubectl apply`.
    

_Lưu ý kỹ thuật quan trọng:_ Khi dùng ArgoCD để triển khai Helm, **ArgoCD chỉ dùng Helm để render ra file YAML**, sau đó tự ArgoCD quản lý trạng thái của các resource đó. Do đó, nếu bạn chạy lệnh `helm list` trên terminal sẽ không thấy release đó xuất hiện — mọi lịch sử phiên bản và thao tác rollback lúc này do ArgoCD nắm giữ hoàn toàn.

## 3. Lợi ích thực tế khi kết hợp bộ đôi này

|**Tiêu chí**|**Chỉ dùng Helm CLI (helm upgrade)**|**Kết hợp ArgoCD + Helm**|
|---|---|---|
|**Theo dõi trạng thái**|Chạy lệnh xong là hết; nếu Pod bị crash hoặc ai đó sửa tay trên cluster thì Helm không tự biết.|Giám sát 24/7 trên Web UI; tự động phát hiện sai lệch (`OutOfSync`) và tự phục hồi (`Self-Heal`).|
|**Quản lý đa môi trường**|Phải chạy nhiều lệnh `helm upgrade -f values-xxx.yaml` thủ công hoặc viết script CI phức tạp.|Dùng **ApplicationSet** của ArgoCD để tự động sinh ra hàng loạt ứng dụng từ danh sách file `values-*.yaml`.|
|**Kiểm soát thay đổi (Audit)**|Khó biết ai đã chạy lệnh `helm upgrade` từ máy cá nhân với tham số gì.|Mọi thay đổi bắt buộc phải đi qua Git commit / Pull Request của file `values.yaml`, có lịch sử rõ ràng.|
|**Bảo mật phân quyền**|Người triển khai cần quyền truy cập trực tiếp vào cụm Kubernetes.|Developer chỉ cần quyền merge PR trên Git; ArgoCD tự kéo về và triển khai an toàn bên trong cụm.|

## 4. Cấu trúc thư mục điển hình trong công ty

Trong kho GitOps của các công ty áp dụng mô hình này, bạn sẽ thường thấy cấu trúc tách bạch rõ ràng giữa **mã nguồn Chart** (hoặc Chart bên ngoài) và **cấu hình môi trường**:

Plaintext

```
gitops-repo/
├── charts/                     # Chứa các Helm Chart nội bộ của công ty
│   └── data-pipeline/
│       ├── Chart.yaml
│       ├── values.yaml         # Giá trị mặc định
│       └── templates/          # Khuôn mẫu Deployment, Service, CronJob...
└── environments/               # Nơi ArgoCD trỏ tới để lấy cấu hình từng môi trường
    ├── dev/
    │   └── data-pipeline-values.yaml
    └── prod/
        └── data-pipeline-values.yaml
```

Khi cần nâng cấp phiên bản ứng dụng lên Production, quy trình CI chỉ cần tự động sửa đúng 1 dòng `tag: "v1.2.5"` trong file `environments/prod/data-pipeline-values.yaml`, và ArgoCD sẽ tự động lo toàn bộ phần còn lại.
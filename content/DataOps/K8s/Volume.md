- Lưu trữ lâu dài, đáng tin cậy cho các pod ngay cả khi pod bị die

- **Volume và Persistent Volume:** Giúp gắn kết dữ liệu bên ngoài Pod, đảm bảo dữ liệu không mất khi Pod bị xoá hoặc tái tạo. Có nhiều loại volume khác nhau: emptyDir, hostPath, configMap, secret, và persistent volume với các kiểu lưu trữ (storage class) đa dạng như NFS, iSCSI, hoặc dịch vụ lưu trữ đám mây như EBS và Azure Disk.

- Có thể gắn địa chỉ bộ nhớ vật lý của ổ cứng lên pod

- Có thể được lưu ở local hoặc remote từ bên ngoài K8s cluster

![[Pasted image 20261103004552.png]]

- Kubernetes sẽ không đảm bảo và chịu trách nhiệm cho volumes, admin phải chủ động sao lưu, sao chép, quản lý

- **Storage Classes và Dynamic Provisioning:** StorageClass định nghĩa kiểu lưu trữ (storage type) mà cluster có thể sử dụng. Cơ chế Dynamic Provisioning (cấp phát động) cho phép Kubernetes tự động tạo Persistent Volume (PV) khi có yêu cầu từ Persistent Volume Claim (PVC), giúp tiết kiệm thời gian và giảm thao tác thủ công cho quản trị viên.
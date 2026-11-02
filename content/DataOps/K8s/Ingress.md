Kubernetes cung cấp khả năng mạng phức tạp hơn nhiều so với chỉ chạy container đơn lẻ. Bạn sẽ học về:

- **Kết nối giữa Pod với Pod:** Mỗi Pod có địa chỉ IP riêng, và các Pod có thể kết nối qua plugin giao diện mạng container (CNI – Container Network Interface) như Calico, Flannel, Weave Net, Cilium.
- **Service và Network Policy:** Khi các Pod thay đổi địa chỉ IP hoặc được tái khởi động, Service đóng vai trò giữ endpoint ổn định cho truy cập và cân bằng tải qua ClusterIP, NodePort, LoadBalancer. Network Policy hoạt động như tường lửa ở layer 3/4 cho phép kiểm soát luồng truy cập giữa các Pod.
- **Ingress Controller:** công cụ điều hướng HTTP/HTTPS từ bên ngoài vào bên trong cluster với tinh năng như:
    - **SSL termination** – chấm dứt mã hóa SSL/TLS ở cấp Ingress để giảm tải cho ứng dụng.
    - **Name-based virtual hosting** – định tuyến theo tên miền.
    - **Path-based routing** – định tuyến theo đường dẫn URL.
    - Các Ingress Controller phổ biến: NGINX, Traefik, HAProxy, Istio Gateway.
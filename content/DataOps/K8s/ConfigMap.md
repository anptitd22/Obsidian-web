![[Pasted image 20261103003707.png]]

- Để chạy một pod thông thường cập nhật cần phải re-build docker -> push docker registry xong pod sẽ pull về và chạy -> rất lâu

- ConfigMap: cấu hình bên ngoài cho ứng dụng, chứa dữ liệu như URL của database, 1 số dịch vụ khác mà bạn sử dụng và chỉ cần connect nó với pod, pod sẽ nhận được dữ liệu mà configmap chứa

- Nếu bạn sửa tên hoặc endpoint của service -> chỉ cần sửa configmap là xong mà không cần build lại

![[Pasted image 20261103003712.png]]
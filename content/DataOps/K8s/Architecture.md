- Một master node - một vài worker node có kubelet (tiến trình kubernetes) chạy trên đó có thể giao tiếp với nhau

- Bên trong các node là các container

- Master node sẽ chạy các tiến trình cần thiết để quản lý cụm

- API server: Chịu trách nhiệm tiếp nhận tất cả yêu cầu RESTful API từ người dùng và các thành phần khác, đóng vai trò là cổng truy cập chính của cluster và là thành phần duy nhất tương tác trực tiếp với etcd.

- Controller Manager = Kiểm tra, theo dõi những gì xảy ra trong cụm (lỗi container, ...). Chạy các controller theo vòng lặp (control loops) để duy trì trạng thái mong muốn:

	- Replication Controller: đảm bảo số lượng bản sao (replica) của ứng dụng luôn đúng.
	- Node Controller: giám sát tình trạng của các node.
	- Endpoint Controller: quản lý danh sách Service endpoints.
	- Service Account & Token Controller: quản lý tài khoản dịch vụ và token truy cập.

- Scheduler: lập lịch các container dựa trên khối lượng công việc và tài nguyên máy chủ có sẵn. 
	- Phân công các Pod chưa được gán lên một node phù hợp dựa trên yêu cầu tài nguyên, ràng buộc (constraints), chính sách ưu tiên (affinity/anti-affinity), vị trí dữ liệu (data locality) và quy tắc lập lịch (scheduling policies).

- etcd: lưu trữ trạng thái hiện tại của cụm kubernetes trong bất cứ thời điểm nào vì nó chứa tất cả dữ liệu cấu hình bên trong tất cả dữ liệu trạng thái của mỗi node và mỗi container trong node đó
	- Lưu trữ dạng key:value
    
    - Nó có thể backup-restore nhờ cơ chế snapshot etcd

![[Pasted image 20261103010355.png]]

- Các node (worker, master) giao tiếp với nhau thông qua virtual network

![[Pasted image 20261103010407.png]]

- Các worker node:

	- higher workload : Chịu tải nặng nhất
	    
	- Ứng dụng bên trong thường nặng hơn và tốn nhiều tài nguyên hơn. Trong khi node master chỉ chạy các tiến trình master nên ít tài nguyên

	- proxy: Quản lý mạng và luật định tuyến để giúp các Service và Pod giao tiếp với nhau hoặc với bên ngoài cluster.

![[Pasted image 20261103010431.png]]

- Node master rất quan trọng nên thường có các bản sao để khôi phục và dự phòng

- Nếu không có master, cụm sẽ không thể truy cập vào được nữa


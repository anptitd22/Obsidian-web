**StatefulSet:** Đối tượng đặc biệt để quản lý các Pod có trạng thái. Mỗi Pod giữ được danh tính riêng và liên kết với volume riêng biệt. Hỗ trợ triển khai (deployment) và xóa (deletion) theo thứ tự.

**Stateful application (Ứng dụng có lưu trạng thái)**:

- Ví dụ: Cơ sở dữ liệu (PostgreSQL, MySQL cluster, MongoDB, Redis cluster, Kafka, ZooKeeper).
    
- Mỗi Pod mang một vai trò và dữ liệu riêng biệt (ví dụ: Node Master/Leader vs Replica/Follower). Pod cần giữ đúng dữ liệu, định danh mạng và thứ tự hoạt động ngay cả khi bị khởi động lại.

![[Pasted image 20261107210854.png]]
- Service: là IP tĩnh, cố định được gắn trên mỗi pod, các pod giao tiếp thông qua service

- Vòng đời của service không liên quan đến pod ngay cả khi pod die

- Các node replication cũng sẽ chung 1 service và service sẽ đóng là một cân bằng tải nó sẽ bắt tín hiệu và chuyển đến node ít bận rộn hơn

![[Pasted image 20261103003458.png]]
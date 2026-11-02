**1. Pod**

- Thành phần cơ bản và nhỏ nhất trong kubernetes

- Lớp trừu tượng của container (docker)
    
- Tạo môi trường hoặc lớp trên cùng container
    
- Có thể thay thế chúng nếu muốn nên người dùng không cần làm việc trực tiếp với chúng (docker, bất kì công nghệ container nào) mà chỉ làm việc với kubernetes
    

- Pod application:

	- Pod application (my-app) chứa ứng dụng chính
	    
	- Pod database với container riêng
	    
	- 1 pod thường chạy 1 container application bên trong
	    
	    - Trường hợp chạy nhiều container: thông thường 1 container chính - một số container phụ (services,...)
        
	- Mỗi pod có một IP riêng có thể giao tiếp nội bộ (không phải container application)
	    
	    - Container application có thể giao tiếp với pod bằng IP đó
	        
	- Pod là tạm thời, có thể hủy do lỗi, chủ động, hết tài nguyên, ...
    
![[Pasted image 20261103003008.png]]

- Nếu pod cũ die, pod mới tạo ra với IP khác dẫn đến phải điều chỉnh container giao tiếp IP mới mỗi khi pod khởi động lại


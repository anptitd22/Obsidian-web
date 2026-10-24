---
title: Docker & Virtual Machine
tags:
  - Docker
  - Virtual Machine
---

![[Compare_docker_VM.png]]

## Docker

- Containers sẽ dùng thẳng OS kernel trên máy dùng
- Ảo hóa hệ điều hành OS
- Ví dụ:
	- Cài 2 hdh ubuntu, archlinux thì khi `uname -a` đều cho ra kernel ubuntu (hdh máy chủ)

## VM

- Sẽ được tạo, quản lý cấp phát tài nguyên hardware nhờ hypervisor, khi đó cài các hdh trên các VM này sẽ tự cài OS kernel riêng 
- Ảo hóa phần cứng

![[VM_architecture.png]]

### Hypervisor

- Có thể được gọi là một VM giám sát (virtual machine monitor - VMN), tạo và điều phối các máy ảo (VM),  là một công nghệ thiết yếu trong cơ sở hạ tầng điện toán hiện đại. Nó giúp cho việc ảo hóa máy tính và máy chủ trở nên khả thi.


|**Tiêu chí**|**Virtual Machine (VM)**|**Docker Container**|
|---|---|---|
|**Cơ chế**|Ảo hóa phần cứng (Hardware-level)|Ảo hóa hệ điều hành (OS-level)|
|**Dung lượng**|**Lớn** (Hàng chục GB do chứa cả Guest OS)|**Rất nhẹ** (Hàng MB đến vài trăm MB)|
|**Khởi động**|**Chậm** (Vài phút để boot hệ điều hành)|**Nhanh** (Vài giây hoặc vài milisecond)|
|**Hiệu năng**|Thấp hơn (Tốn tài nguyên CPU/RAM cho Guest OS)|Gần như nguyên bản (Native performance)|
|**Mức độ cô lập**|**Rất cao** (Tách biệt hoàn toàn ở cấp phần cứng)|**Vừa phải** (Tách biệt ở cấp process/namespace)|
|**Tính di động**|Phụ thuộc vào Hypervisor|Chạy ở bất kỳ đâu có Docker Engine|

#### Nên chọn Docker khi:

- Phát triển ứng dụng microservices hoặc web app hiện đại.
    
- Cần triển khai nhanh (CI/CD pipeline), mở rộng (scale) hàng chục/hàng trăm instance trong vài giây.
    
- Muốn tiết kiệm tối đa tài nguyên máy chủ (RAM, CPU).
    
- Cần đảm bảo môi trường ở máy dev giống hệt máy production ("Chạy được trên máy tôi thì chạy được trên server").
    
#### Nên chọn VM khi:

- Cần chạy các ứng dụng yêu cầu hệ điều hành hoàn toàn khác nhau trên cùng một server (ví dụ: Chạy ứng dụng Windows Server trên host Linux).
    
- Cần mức độ **bảo mật và cô lập tuyệt đối** (khi chạy ứng dụng của nhiều khách hàng khác nhau - multi-tenant).
    
- Chạy các ứng dụng monolithic cũ (Legacy apps) đòi hỏi cấu hình hệ thống hoặc driver phần cứng sâu.
---
title: Architecture
tags:
  - Docker
---

![[Pasted image 20261024231058.png]]

### Docker Host

Docker Host là máy vật lý hoặc máy ảo của bạn chạy hệ điều hành Linux (hoặc hệ điều hành khác tương thích với Docker Engine). Nó có thể nằm trên cùng một máy vật lý với máy bạn đang chạy Docker Engine hoặc trên các máy ảo hoặc máy vật lý khác.

### Docker Engine

Docker Engine là một công nghệ container hóa mã nguồn mở dùng để xây dựng và đóng gói ứng dụng của bạn. Docker Engine hoạt động như một ứng dụng máy chủ-máy khách với:

- Một máy chủ với tiến trình nền chạy liên tục `dockerd`.

- Các API quy định giao diện mà các chương trình có thể sử dụng để giao tiếp và hướng dẫn Docker deamon.

- Một giao diện dòng lệnh (CLI) dành cho Docker.

Giao diện dòng lệnh (CLI) sử dụng các API của Docker để điều khiển hoặc tương tác với Docker deamon thông qua kịch bản hoặc các lệnh CLI trực tiếp. Nhiều ứng dụng Docker khác cũng sử dụng API và CLI cơ bản này. Tiến trình nền (deamon) tạo và quản lý các đối tượng Docker, chẳng hạn như image, container, network và volume.

### Docker Daemon

Về cơ bản, Docker Host đang chạy Docker Daemon. Daemon này lắng nghe các yêu cầu API của client như “docker run”, “docker builds”, và quản lý các đối tượng Docker như image, container, network, v.v. như một phần công việc của nó. Ngoài ra, để quản lý các dịch vụ Docker, một daemon có thể dễ dàng giao tiếp với các daemon khác.


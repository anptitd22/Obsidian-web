---
title: Network
tags:
  - Docker
---

Docker's networking subsystem is pluggable, using drivers. Có một số trình điều khiển mặc định, cung cấp các chức năng mạng cốt lõi:

- `bridge`: Trình điều khiển mạng mặc định. Mạng bridge thường được sử dụng khi ứng dụng của bạn chạy trong một container cần giao tiếp với các container khác trên cùng một máy chủ. Xem [Bridge network driver](https://docs.docker.com/engine/network/drivers/bridge/).

- `host`: Loại bỏ sự cách ly mạng giữa container và máy chủ Docker, và sử dụng trực tiếp mạng của máy chủ. Xem [Host network driver](https://docs.docker.com/engine/network/drivers/host/).

- `Overlay`: Mạng lớp phủ kết nối nhiều tiến trình Docker với nhau và cho phép các dịch vụ và container của Swarm giao tiếp giữa các node. Chiến lược này loại bỏ nhu cầu thực hiện định tuyến ở cấp độ hệ điều hành. Xem trình [Overlay network driver](https://docs.docker.com/engine/network/drivers/overlay/).

- `ipvlan`: Mạng IPvlan cung cấp cho người dùng quyền kiểm soát hoàn toàn cả địa chỉ IPv4 và IPv6. Trình điều khiển VLAN được xây dựng trên nền tảng đó, cung cấp cho người vận hành quyền kiểm soát hoàn toàn việc gắn thẻ VLAN lớp 2 và thậm chí cả định tuyến IPvlan L3 cho những người dùng quan tâm đến việc tích hợp mạng lớp dưới. Xem [IPvlan network driver](https://docs.docker.com/engine/network/drivers/ipvlan/).

- `macvlan`: Mạng Macvlan cho phép bạn gán địa chỉ MAC cho một container, làm cho nó xuất hiện như một thiết bị vật lý trên mạng của bạn. Trình nền Docker định tuyến lưu lượng truy cập đến các container dựa trên địa chỉ MAC của chúng. Sử dụng trình điều khiển macvlan đôi khi là lựa chọn tốt nhất khi xử lý các ứng dụng cũ mong muốn được kết nối trực tiếp với mạng vật lý, thay vì được định tuyến thông qua stack mạng của máy chủ Docker. Xem [Macvlan network driver](https://docs.docker.com/engine/network/drivers/macvlan/).

- `none`: Cách ly hoàn toàn một container khỏi máy chủ và các container khác. Tùy chọn none không khả dụng cho các dịch vụ Swarm. Xem [None network driver](https://docs.docker.com/engine/network/drivers/none/).

- [Network plugins](https://docs.docker.com/engine/extend/plugins_network/): Bạn có thể cài đặt và sử dụng các plugin mạng của bên thứ ba với Docker.

## Bridge network

A Docker bridge network có một IPv4 subnet và, tùy chọn, một mạng IPv6 subnet. Mỗi container được kết nối với bridge network đều có một giao diện mạng với các địa chỉ trong các mạng con của mạng đó. Theo mặc định:

- Cho phép truy cập mạng không hạn chế vào các container trong mạng từ host và từ các container khác được kết nối với cùng một bridge network.

- Chặn truy cập từ các container trong mạng khác và từ bên ngoài máy chủ Docker.

- Sử dụng kỹ thuật che giấu địa chỉ IP để cấp quyền truy cập mạng bên ngoài cho các container. Các thiết bị trên mạng bên ngoài của máy chủ chỉ thấy địa chỉ IP của máy chủ Docker.

- Hỗ trợ công bố cổng (export, reverse), trong đó lưu lượng mạng được chuyển tiếp giữa các cổng của container và các cổng trên địa chỉ IP của máy chủ. Các cổng đã công bố có thể được truy cập từ bên ngoài máy chủ Docker, thông qua địa chỉ IP của nó.

Port: `<externel host>:<internal container>`

## Host network

Nếu bạn sử dụng chế độ mạng `host` cho một container, ngăn xếp mạng của container đó không được cách ly khỏi máy chủ Docker (container chia sẻ không gian tên mạng của máy chủ), và container không được cấp địa chỉ IP riêng. Ví dụ, nếu bạn chạy một container liên kết với cổng 80 và bạn sử dụng mạng `host`, ứng dụng của container sẽ khả dụng trên cổng 80 với địa chỉ IP của máy chủ.
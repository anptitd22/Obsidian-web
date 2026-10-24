---
title: Docker Storage
tags:
  - Docker
---

## Volume

Volume là kho lưu trữ dữ liệu bền vững cho các container, được tạo và quản lý bởi Docker. Bạn có thể tạo volume một cách rõ ràng bằng lệnh `docker volume create`, hoặc Docker có thể tạo volume trong quá trình tạo container hoặc service.

Khi bạn tạo một volume, nó được lưu trữ trong một thư mục trên máy chủ Docker. Khi bạn gắn volume đó vào một container, thư mục này chính là thứ được gắn vào container. Điều này tương tự như cách hoạt động của bind mount, ngoại trừ việc volume được quản lý bởi Docker và được tách biệt khỏi chức năng cốt lõi của máy chủ.

Volumes là cơ chế được ưu tiên để lưu trữ dữ liệu được tạo ra và sử dụng bởi các container Docker. Trong khi bind mounts phụ thuộc vào cấu trúc thư mục và hệ điều hành của máy chủ, volumes được quản lý hoàn toàn bởi Docker. Volumes là lựa chọn tốt cho các trường hợp sử dụng sau:

- Việc sao lưu hoặc di chuyển các volume dễ dàng hơn so với việc sử dụng bind mount.
    
- Bạn có thể quản lý dung lượng lưu trữ bằng các lệnh CLI của Docker hoặc API của Docker.
    
- Volumes hoạt động trên cả container Linux và Windows.
    
- Volumes có thể được chia sẻ an toàn hơn giữa nhiều container.
    
- New volumes có thể được điền sẵn nội dung bằng container hoặc build.
    
- Khi ứng dụng của bạn yêu cầu hiệu năng I/O cao.

>[!note]
>User có thể dùng quyền admin hay `sudo` để truy cập vào volume

### Lưu ý

Volume không phải là lựa chọn tốt nếu bạn cần truy cập các tệp từ máy chủ, vì volume được quản lý hoàn toàn bởi Docker. Hãy sử dụng bind mount nếu bạn cần truy cập các tệp hoặc thư mục từ cả container và máy chủ.

Việc sử dụng volume thường tốt hơn so với việc ghi dữ liệu trực tiếp vào container, bởi vì volume không làm tăng kích thước của các container sử dụng nó. Sử dụng volume cũng nhanh hơn; việc ghi vào container's writable layer yêu cầu trình điều khiển lưu trữ([storage driver](https://docs.docker.com/engine/storage/drivers/)) để quản lý hệ thống tệp. The storage driver cung cấp một union filesystem, sử dụng nhân Linux. Lớp trừu tượng bổ sung này làm giảm hiệu suất so với việc sử dụng volume, vốn ghi trực tiếp vào hệ thống tệp của máy chủ.

Nếu container của bạn tạo ra dữ liệu trạng thái không bền vững, hãy cân nhắc sử dụng mount tmpfs để tránh lưu trữ dữ liệu ở bất kỳ đâu vĩnh viễn và để tăng hiệu suất của container bằng cách tránh ghi vào the container's writable layer.

Các volume sử dụng cơ chế truyền liên kết rprivate (riêng tư đệ quy), và cơ chế truyền liên kết này không thể cấu hình được đối với các volume.

### Vòng đời

Nội dung của một volume tồn tại bên ngoài vòng đời của một container nhất định. Khi một container bị hủy, lớp ghi (writable layer) cũng bị hủy theo. Việc sử dụng volume đảm bảo rằng dữ liệu được lưu giữ ngay cả khi container sử dụng nó bị xóa.

Một volume nhất định có thể được gắn vào nhiều container cùng lúc. Khi không có container nào đang chạy sử dụng volume đó, volume vẫn khả dụng cho Docker và không bị xóa tự động. Bạn có thể xóa các volume không sử dụng bằng lệnh `docker volume prune`.

## Bind mount

Khi sử dụng bind mount, một tệp hoặc thư mục trên máy chủ được gắn từ máy chủ vào container. Ngược lại, khi sử dụng volume, một thư mục mới được tạo trong thư mục lưu trữ của Docker trên máy chủ, và Docker quản lý nội dung của thư mục đó.

Dùng khi:

- Chia sẻ mã nguồn hoặc các tệp tạo lập giữa môi trường phát triển trên máy chủ Docker và container.
    
- Khi bạn muốn tạo hoặc tạo ra các tệp trong một container và lưu trữ các tệp đó vào hệ thống tệp của máy chủ.
    
- Chia sẻ các tệp cấu hình từ máy chủ đến các container. Đây là cách Docker cung cấp khả năng phân giải DNS cho các container theo mặc định, bằng cách mount tệp `/etc/resolv.conf` từ máy chủ vào mỗi container.

Tính năng gắn kết thư mục (bind mount) cũng có sẵn cho quá trình builds: bạn có thể gắn kết mã nguồn từ máy chủ vào build container để kiểm tra, kiểm tra cú pháp hoặc biên dịch dự án.

### Considerations and constraints

Theo mặc định, các mount liên kết có quyền ghi vào các tệp trên máy chủ.

Một tác dụng phụ của việc sử dụng bind mount là bạn có thể thay đổi hệ thống tập tin của máy chủ thông qua các tiến trình chạy trong container, bao gồm việc tạo, sửa đổi hoặc xóa các tập tin hoặc thư mục hệ thống quan trọng. Khả năng này có thể gây ra những vấn đề về bảo mật. Ví dụ, nó có thể ảnh hưởng đến các tiến trình không phải Docker trên hệ thống máy chủ.

Bạn có thể sử dụng tùy chọn `readonly` hoặc `ro` để ngăn container ghi dữ liệu vào thư mục được mount.

Các điểm gắn kết (bind mount) được tạo trên máy chủ Docker daemon, chứ không phải trên máy khách - client.

Nếu bạn đang sử dụng remote Docker deamon, bạn không thể tạo blind mount để truy cập các tệp trên máy khách-client trong container.

Đối với Docker Desktop, tiến trình nền (daemon) chạy bên trong máy ảo Linux, chứ không phải trực tiếp trên máy chủ gốc. Docker Desktop có các cơ chế tích hợp sẵn giúp xử lý việc mount thư mục một cách minh bạch, cho phép bạn chia sẻ đường dẫn hệ thống tệp của máy chủ gốc với các container chạy trong máy ảo.

Các container có bind mount được liên kết chặt chẽ với máy chủ.

Việc gắn kết thư mục (bind mount) phụ thuộc vào việc hệ thống tệp của máy chủ có cấu trúc thư mục cụ thể hay không. Điều này có nghĩa là các container sử dụng gắn kết thư mục có thể bị lỗi nếu chạy trên một máy chủ khác không có cùng cấu trúc thư mục.

## tmpfs mounts

Volumes và bind mounts cho phép bạn chia sẻ tập tin giữa máy chủ và container để dữ liệu có thể được lưu giữ ngay cả sau khi container dừng hoạt động.

Nếu bạn đang chạy Docker trên Linux, bạn có thêm một lựa chọn thứ ba: tmpfs mount. Khi bạn tạo một container với tmpfs mount, container đó có thể tạo các tệp bên ngoài container's writable layer.

Khác với các volume và bind mount, tmpfs mount chỉ là tạm thời và chỉ được lưu trữ trong bộ nhớ của máy chủ. Khi container dừng hoạt động, tmpfs mount sẽ bị xóa và các tệp được ghi vào đó sẽ không được lưu giữ.

Việc sử dụng tmpfs (TPF) là lựa chọn tốt nhất trong trường hợp bạn không muốn dữ liệu được lưu trữ lâu dài trên máy chủ hoặc bên trong container. Điều này có thể vì lý do bảo mật hoặc để bảo vệ hiệu năng của container khi ứng dụng của bạn cần ghi một lượng lớn dữ liệu trạng thái không được lưu trữ lâu dài.

>[!warning]
>Các điểm tmpfs mount trong Docker ánh xạ trực tiếp đến tmpfs trong nhân Linux. Do đó, dữ liệu tạm thời có thể được ghi vào tệp hoán đổi (swap file), và nhờ vậy được lưu trữ lâu dài trên hệ thống tệp.

Liên quan:

- [[Docker Compose]]

- [[Docker File]]
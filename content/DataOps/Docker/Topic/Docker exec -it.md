---
title: Docker exect -it
tags:
  - Docker
---

- **`docker exec`**: Lệnh yêu cầu Docker thực thi một tiến trình mới bên trong một container đang chạy (chứ không phải tạo mới container).
    
- **`-i` (`--interactive`)**: Giữ cổng đầu vào chuẩn (STDIN) mở, cho phép bạn gõ và gửi lệnh vào container.
    
- **`-t` (`--tty`)**: Cấp một thiết bị đầu cuối ảo (pseudo-TTY), giúp hiển thị giao diện dòng lệnh đầy đủ với màu sắc, tự động xuống dòng và xử lý các phím tắt (như Ctrl+C).
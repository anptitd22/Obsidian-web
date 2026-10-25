---
title: Architecture
tags:
  - MySQL
---
## Overview

![[Pasted image 20261023233614.png]]

Hỗ trợ nhiều Storage Engine như InnoDB(mặc định), MyISAM, Memory, ...

MySQL sử dụng mô hình **Luồng (Thread-per-connection)** hoặc **Thread Pool**. Toàn bộ MySQL Server chạy trong **một tiến trình (Single OS Process: `mysqld`)** duy nhất

- **Ưu điểm**: Chi phí chuyển đổi ngữ cảnh (_context switch_) thấp hơn, tiêu tốn ít RAM cho mỗi kết nối hơn PostgreSQL.
    
- **Nhược điểm**: Nếu một luồng gặp lỗi bộ nhớ nghiêm trọng (như segfault do bug engine), toàn bộ tiến trình `mysqld` có thể sập (crash toàn bộ instance).

## InnoDB

![[Pasted image 20261024003623.png]]

Mỗi 1 bảng (không phân vùng) tương ứng với 1 file .ibd ( bao gồm dữ liệu và index)

Dữ liệu được tìm kiếm trên buffer pool trước nếu miss thì xuống disk để đọc
---
title: Docker Image
tags:
  - Docker
---

Là một bản thiết kế của container chứa các mã, thư viện, biến môi trường,.. cần thiết để chạy một application

## Image Base

Image Base là nền tảng để xây dựng các image khác. Có thể sử dụng bất kỳ image nào làm image base. Tuy nhiên, một số image được tạo ra có chủ đích như những khối xây dựng, cung cấp nền tảng hoặc điểm khởi đầu cho một ứng dụng.

```dockerfile
FROM alpine:latest AS downloader
```

## Layer

Khi chạy dockerfile, mỗi construction (câu lệnh) sẽ tạo một layer, có khả năng tái sử dụng và mở rộng,

- Mỗi lớp đại diện cho sự thay đổi của hệ thống (thêm, sửa, xóa)

- Cấu tạo từ các layer bất biến, khi thay đổi image docker sẽ thêm 1 layer khác ở bên trên nó

- Layer cuối là lớp có thể ghi, được tạo khi chạy 1 image

![[Pasted image 20261024234522.png]]

- Có thể tận dụng layer cho nhiều image

![[Pasted image 20261024234541.png]]

- Các bước chạy layer sẽ được lưu vào cache (disk)

- Là chìa khóa để thiết kế một image nhanh, gọn

Ví dụ:

```Dockerfile
FROM alpine:latest AS downloader

ARG POSTGRESQL_CONNECTOR_VERSION=42.7.4

RUN apk add --no-cache curl

WORKDIR /downloads

RUN curl -L -o postgresql-${POSTGRESQL_CONNECTOR_VERSION}.jar \
[https://repo1.maven.org/maven2/org/postgresql/postgresql/${POSTGRESQL_CONNECTOR_VERSION}/postgresql-${POSTGRESQL_CONNECTOR_VERSION}.jar](https://repo1.maven.org/maven2/org/postgresql/postgresql/$%7BPOSTGRESQL_CONNECTOR_VERSION%7D/postgresql-$%7BPOSTGRESQL_CONNECTOR_VERSION%7D.jar)
```

- Dockerfile trên sẽ tạo ra 4 layer và 1 layer image cuối (lớp ghi- writable layer) (alpine là image base)

## Union filesystem

![[Pasted image 20261024235048.png]]

Có tên gọi khác là OverlayFS

Khi một container được chạy nó sẽ sử dụng union filesystem terminology để hợp nhất các layer trên filesystem được docker sử dụng -> cho phép tái sử dụng các layer khác nhau, có thể xem sự thay đổi thông qua câu lệnh `docker inspect <container ID>`

- Lower Directories: các file layer từ image
    
- Upper Directory: là không gian duy nhất dành riêng cho các container đó, cho phép một image cho thể sử dụng cho nhiều container và một sự thay đổi của một container sẽ không được thấy bởi container khác
    
- Merged Directory: các file layer sau khi merge

Liên quan:

- [[Docker Container]]

- [[Docker File]]

- [[Docker Registry]]

- [[Docker Compose]]

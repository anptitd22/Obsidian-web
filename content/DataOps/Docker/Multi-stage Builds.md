---
title: Multi-stage Builds
tags:
  - Docker
---

Xây dựng nhiều giai đoạn rất hữu ích cho bất kỳ ai đã từng gặp khó khăn trong việc tối ưu hóa Dockerfile mà vẫn đảm bảo tính dễ đọc và dễ bảo trì.

Dockerfile sau đây có hai giai đoạn riêng biệt: một giai đoạn để xây dựng tệp nhị phân, và một giai đoạn khác để sao chép tệp nhị phân từ giai đoạn đầu tiên sang giai đoạn tiếp theo.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.25
WORKDIR /src
COPY <<EOF ./main.go
package main

import "fmt"

func main() {
  fmt.Println("hello, world")
}
EOF
RUN go build -o /bin/hello ./main.go

FROM scratch
COPY --from=0 /bin/hello /bin/hello
CMD ["/bin/hello"]
```

Bạn chỉ cần một Dockerfile duy nhất. Không cần tập lệnh xây dựng riêng biệt. Chỉ cần chạy lệnh `docker build`.

```console
 docker build -t hello .
```

Kết quả cuối cùng là một production image nhỏ gọn chỉ chứa mỗi mã nhị phân. Không có công cụ xây dựng nào cần thiết như golang, ... để biên dịch ứng dụng được bao gồm trong image kết quả.

Nó hoạt động như thế nào? 

Lệnh `FROM` thứ hai bắt đầu một giai đoạn xây dựng mới với image base làm cơ sở. Dòng `COPY --from=0` chỉ sao chép sản phẩm đã được xây dựng từ giai đoạn trước vào giai đoạn mới này. Go SDK và bất kỳ sản phẩm trung gian nào khác sẽ bị giữ lại và không được lưu trong image cuối cùng.

## Name your build stages

Theo mặc định, các giai đoạn không được đặt tên và bạn tham chiếu chúng bằng số nguyên, bắt đầu từ `0` cho lệnh `FROM` đầu tiên. Tuy nhiên, bạn có thể đặt tên cho các giai đoạn bằng cách thêm `AS <NAME>` vào lệnh `FROM`. Ví dụ này cải tiến ví dụ trước bằng cách đặt tên cho các giai đoạn và sử dụng tên đó trong lệnh `COPY`. Điều này có nghĩa là ngay cả khi các lệnh trong Dockerfile của bạn được sắp xếp lại sau này, lệnh `COPY` vẫn không bị lỗi.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.25 AS build
WORKDIR /src
COPY <<EOF /src/main.go
package main

import "fmt"

func main() {
  fmt.Println("hello, world")
}
EOF
RUN go build -o /bin/hello ./main.go

FROM scratch
COPY --from=build /bin/hello /bin/hello
CMD ["/bin/hello"]
```

## Stop at a specific build stage

Khi xây dựng image, bạn không nhất thiết phải xây dựng toàn bộ Dockerfile bao gồm mọi giai đoạn. Bạn có thể chỉ định giai đoạn xây dựng mục tiêu. Lệnh sau giả định bạn đang sử dụng Dockerfile trước đó nhưng dừng lại ở giai đoạn có tên là `build`:

```console
 docker build --target build -t hello .
```

Một vài trường hợp mà điều này có thể hữu ích là:

- Gỡ lỗi một giai đoạn xây dựng cụ thể

- Sử dụng `debug` stage với tất cả các ký hiệu hoặc công cụ gỡ lỗi được bật, và một `production` stage tinh gọn.

- Sử dụng `testing` stage để nạp dữ liệu thử nghiệm cho ứng dụng, nhưng xây dựng phiên bản chính thức bằng một giai đoạn khác sử dụng dữ liệu thực.

## Use an external image as a stage

Khi sử dụng multi-stage builds, bạn không bị giới hạn chỉ sao chép từ các giai đoạn đã tạo trước đó trong Dockerfile. Bạn có thể sử dụng lệnh `COPY --from` để sao chép từ một image riêng biệt, bằng cách sử dụng tên image cục bộ, tag có sẵn cục bộ hoặc trên kho lưu trữ Docker, hoặc ID tag. Máy khách Docker sẽ tải image nếu cần và sao chép tệp từ đó. Cú pháp như sau:

```dockerfile
COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf
```

## Use a previous stage as a new stage

Bạn có thể tiếp tục từ điểm mà giai đoạn trước đã dừng lại bằng cách tham chiếu đến giai đoạn đó khi sử dụng chỉ thị `FROM`. Ví dụ:

```dockerfile
# syntax=docker/dockerfile:1

FROM alpine:latest AS builder
RUN apk --no-cache add build-base

FROM builder AS build1
COPY source1.cpp source.cpp
RUN g++ -o /binary source.cpp

FROM builder AS build2
COPY source2.cpp source.cpp
RUN g++ -o /binary source.cpp
```

## Differences between legacy builder and BuildKit

Trình biên dịch Docker Engine cũ xử lý tất cả các tage của Dockerfile dẫn đến target stage đã chọn `--target`. Nó sẽ biên dịch một giai đoạn ngay cả khi target stage không phụ thuộc vào tage đó.

BuildKit chỉ xây dựng các giai đoạn mà target stage phụ thuộc vào.

Ví dụ, với Dockerfile sau:

```dockerfile
# syntax=docker/dockerfile:1
FROM ubuntu AS base
RUN echo "base"

FROM base AS stage1
RUN echo "stage1"

FROM base AS stage2
RUN echo "stage2"
```

Khi BuildKit được bật, việc xây dựng mục tiêu stage2 trong Dockerfile này có nghĩa là chỉ có base và stage2 được xử lý. Không có sự phụ thuộc vào stage1, vì vậy nó bị bỏ qua.

```console
 DOCKER_BUILDKIT=1 docker build --no-cache -f Dockerfile --target stage2 .
[+] Building 0.4s (7/7) FINISHED                                                                    
 => [internal] load build definition from Dockerfile                                            0.0s
 => => transferring dockerfile: 36B                                                             0.0s
 => [internal] load .dockerignore                                                               0.0s
 => => transferring context: 2B                                                                 0.0s
 => [internal] load metadata for docker.io/library/ubuntu:latest                                0.0s
 => CACHED [base 1/2] FROM docker.io/library/ubuntu                                             0.0s
 => [base 2/2] RUN echo "base"                                                                  0.1s
 => [stage2 1/1] RUN echo "stage2"                                                              0.2s
 => exporting to image                                                                          0.0s
 => => exporting layers                                                                         0.0s
 => => writing image sha256:f55003b607cef37614f607f0728e6fd4d113a4bf7ef12210da338c716f2cfd15    0.0s
```

Mặt khác, việc xây dựng mục tiêu tương tự mà không sử dụng BuildKit sẽ dẫn đến việc xử lý tất cả các giai đoạn:

```console
 DOCKER_BUILDKIT=0 docker build --no-cache -f Dockerfile --target stage2 .
Sending build context to Docker daemon  219.1kB
Step 1/6 : FROM ubuntu AS base
 ---> a7870fd478f4
Step 2/6 : RUN echo "base"
 ---> Running in e850d0e42eca
base
Removing intermediate container e850d0e42eca
 ---> d9f69f23cac8
Step 3/6 : FROM base AS stage1
 ---> d9f69f23cac8
Step 4/6 : RUN echo "stage1"
 ---> Running in 758ba6c1a9a3
stage1
Removing intermediate container 758ba6c1a9a3
 ---> 396baa55b8c3
Step 5/6 : FROM base AS stage2
 ---> d9f69f23cac8
Step 6/6 : RUN echo "stage2"
 ---> Running in bbc025b93175
stage2
Removing intermediate container bbc025b93175
 ---> 09fc3770a9c4
Successfully built 09fc3770a9c4
```

Trình xây dựng hệ thống kế thừa xử lý giai đoạn 1, ngay cả khi giai đoạn 2 không phụ thuộc vào nó.

Liên quan:

- [[Docker File]]

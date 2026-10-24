---
title: Docker File
tags:
  - Docker
---

Là một bản thiết kế của container, là một công thức build docker image, là snapshot mà application được đóng gói cùng với các dependence của tệp.

## Câu lệnh

Một số lệnh trên Dockerfile hay dùng:

- `FROM <image>` - this specifies the base image that the build will extend.
    
- `WORKDIR <path> `- this instruction specifies the "working directory" or the path in the image where files will be copied and commands will be executed.
    
- `COPY <host-path> <image-path>` - this instruction tells the builder to copy files from the host and put them into the container image.
    
- `RUN` <command> - this instruction tells the builder to run the specified command.
    
- `ENV <name> <value> `- this instruction sets an environment variable that a running container will use.
    
- `EXPOSE <port-number>` - this instruction sets configuration on the image that indicates a port the image would like to expose.
    
- `USER <user-or-uid>` - this instruction sets the default user for all subsequent instructions.
    
- `CMD ["<command>", "<arg1>"]` - this instruction sets the default command a container using this image will run.

Nhanh chóng đóng gói các dự án mới vào container với `docker init`.

Sau khi build có thể tag, publish lên docker registry

## Cache

- Bất kỳ thay đổi nào đối với các tệp được sao chép vào image bằng lệnh COPY hoặc ADD. Docker theo dõi mọi thay đổi đối với các tệp trong thư mục dự án của bạn. Cho dù đó là thay đổi về nội dung hay các thuộc tính như quyền truy cập, Docker đều coi những sửa đổi này là tác nhân để vô hiệu hóa bộ nhớ cache.

- Khi một lớp bị vô hiệu hóa, tất cả các lớp tiếp theo cũng sẽ bị vô hiệu hóa. Nếu bất kỳ layer nào trước đó, bao gồm cả image base hoặc các layer trung gian, đã bị vô hiệu hóa do thay đổi, Docker sẽ đảm bảo rằng các layer tiếp theo phụ thuộc vào nó cũng sẽ bị vô hiệu hóa. Điều này giúp quá trình xây dựng được đồng bộ và ngăn ngừa sự không nhất quán.

- Nên --no-cache cho các câu lệnh `curl` vì nó sẽ làm nặng image đôi khi các file tải về đã xóa mà muốn tải lại nhưng do kết quả lưu ở cache nên nó sẽ bỏ qua bước đó.

Liên quan:

- [[Docker Image]]

- [[Multi-stage Builds]]

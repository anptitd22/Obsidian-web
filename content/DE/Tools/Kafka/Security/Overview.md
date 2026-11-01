Xác thực các kết nối đến các broker từ các client (nhà sản xuất và người tiêu dùng), các broker và công cụ khác, bằng cách sử dụng SSL hoặc SASL. Kafka hỗ trợ các cơ chế SASL sau:

- SASL/GSSAPI (Kerberos) - starting at version 0.9.0.0
- SASL/PLAIN - starting at version 0.10.0.0
- SASL/SCRAM-SHA-256 and SASL/SCRAM-SHA-512 - starting at version 0.10.2.0
- SASL/OAUTHBEARER - starting at version 2.0

Mã hóa dữ liệu được truyền giữa các brokers và clients, giữa các brokers, hoặc giữa các brokers và các công cụ bằng SSL (Lưu ý rằng sẽ có sự suy giảm hiệu năng khi bật SSL, mức độ suy giảm phụ thuộc vào loại CPU và cách triển khai JVM).

Ủy quyền (Authorization) các thao tác đọc/ghi của client

Chức năng authorization có thể cắm thêm và hỗ trợ tích hợp với các dịch vụ authorization bên ngoài.

>Cần lưu ý rằng bảo mật là tùy chọn



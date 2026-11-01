### 1. Cấu trúc Message (Message Structure)

- **Header:** Có độ dài thay đổi (_variable length_).
    
- **Key:** Mảng byte thuần túy/mờ (_opaque byte array_) có độ dài thay đổi.
    
- **Value:** Mảng byte thuần túy/mờ (_opaque byte array_) có độ dài thay đổi.
    

### 2. Triết lý thiết kế: Key & Value dạng Opaque (Byte array)

- **Nguyên nhân:** Các thư viện tuần tự hóa dữ liệu (_serialization_) phát triển liên tục và đa dạng (Avro, Protobuf, JSON, Schema Registry...); không có một chuẩn duy nhất nào phù hợp cho mọi use-case.
    
- **Lợi ích:**
    
    - Kafka Broker hoạt động độc lập, không cần quan tâm hay ép buộc định dạng dữ liệu bên trong payload.
        
    - Việc lựa chọn cơ chế và định dạng serialization cụ thể được chuyển giao hoàn toàn cho phía ứng dụng client (Producer/Consumer) tự quyết định.
        

### 3. Giao diện `RecordBatch`

- **Bản chất:** Đóng vai trò như một iterator lặp qua tập hợp các thông điệp.
    
- **Chức năng chính:** Cung cấp các phương thức chuyên dụng để tối ưu hóa việc đọc và ghi dữ liệu hàng loạt (_batching_) trực tiếp qua các Kênh NIO (_Java NIO Channels_).
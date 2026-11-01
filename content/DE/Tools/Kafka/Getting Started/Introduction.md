## 1. Event Streaming là gì?

- **Khái niệm:** Dòng chảy dữ liệu thời gian thực (như hệ thần kinh trung ương của cơ thể), thu thập, lưu trữ, xử lý và định tuyến dữ liệu từ các nguồn (database, sensor, mobile, cloud) đến các đích đến một cách liên tục.
    
- **Mục đích:** Đảm bảo đúng thông tin, đúng nơi, đúng thời điểm trong thế giới tự động hóa và luôn trực tuyến.
    

## 2. Ứng dụng của Event Streaming

- Xử lý thanh toán và giao dịch tài chính (ngân hàng, chứng khoán, bảo hiểm).
    
- Theo dõi xe cộ, vận chuyển và logistics theo thời gian thực.
    
- Thu thập và xử lý dữ liệu cảm biến IoT (nhà máy, năng lượng gió).
    
- Phản ứng tức thì với tương tác và đơn hàng của khách hàng (bán lẻ, du lịch, app di động).
    
- Giám sát bệnh nhân y tế và dự báo tình trạng khẩn cấp.
    
- Kết nối dữ liệu giữa các phòng ban trong doanh nghiệp.
    
- Làm nền tảng cho data platform, event-driven architecture và microservices.
    

## 3. Apache Kafka là gì?

Là nền tảng event streaming phân tán, tích hợp 3 khả năng chính:

1. **Publish & Subscribe:** Đọc/ghi luồng event và tích hợp dữ liệu từ hệ thống khác.
    
2. **Lưu trữ bền vững & đáng tin cậy:** Giữ lại các luồng event trong thời gian tùy chọn.
    
3. **Xử lý luồng event:** Xử lý dữ liệu ngay khi xảy ra hoặc hồi tố (retrospectively).
    

- **Đặc điểm:** Khả năng mở rộng cao, đàn hồi, chịu lỗi (fault-tolerant), bảo mật; chạy được trên bare-metal, VM, container, on-premises hay cloud.
    

## 4. Kiến trúc trong một nỏm (How Kafka Works)

- **Servers (Brokers & Connect):** Cụm server (cluster) đa trung tâm dữ liệu. Một số làm tầng lưu trữ (brokers), số khác chạy Kafka Connect để tích hợp dữ liệu với hệ thống ngoài. Có khả năng chịu lỗi cao (server này hỏng, server khác tự động thay thế).
    
- **Clients:** Thư viện/ứng dụng đọc-ghi-xử lý event song song, quy mô lớn (hỗ trợ Java, Scala, Go, Python, C/C++, REST APIs...).
    

## 5. Các Khái niệm & Thuật ngữ Chính (Main Concepts)

- **Event (Record / Message):** Ghi lại một sự kiện "đã xảy ra" (gồm: _key, value, timestamp, headers_).
    
- **Producers & Consumers:** Ứng dụng client ghi (produce) và đọc/xử lý (consume) event. Hoàn toàn tách biệt nhau để đảm bảo khả năng mở rộng.

![[Pasted image 20261101160333.png]]

- **Topics:** Nơi tổ chức và lưu trữ bền vững các event (tương tự thư mục). Event không bị xóa ngay sau khi đọc mà được giữ lại theo cấu hình thời gian của topic.
    
- **Partitions (Phân vùng):** Topic được chia thành nhiều partition nằm trên các broker khác nhau giúp mở rộng quy mô. Event có cùng `key` sẽ vào cùng một partition và được đọc đúng thứ tự.
    
- **Replication (Sao chép):** Mỗi partition được sao chép sang nhiều broker khác nhau (thường hệ số sao chép là 3) để đảm bảo tính sẵn sàng cao và chống mất dữ liệu.
    

## 6. Các Kafka APIs Cốt Lõi (Dành cho Java & Scala)

- **Admin API:** Quản lý và kiểm tra topic, broker.
    
- **Producer API:** Ghi luồng event vào topic.
    
- **Consumer API:** Đọc và xử lý luồng event từ topic.
    
- **Kafka Streams API:** Xây dựng ứng dụng xử lý luồng phức tạp (biến đổi, tổng hợp, kết hợp, xử lý theo thời gian).
    
- **Kafka Connect API:** Xây dựng trình kết nối nhập/xuất dữ liệu tái sử dụng với các hệ thống bên ngoài (như cơ sở dữ liệu).
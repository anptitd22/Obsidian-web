## 1. Tổng quan về Giám sát Kafka (Overview)

- **Khái niệm:** Giám sát Kafka (Kafka monitoring) là quá trình theo dõi và phân tích liên tục hiệu năng cũng như hành vi vận hành của cụm (cluster) Apache Kafka.
- **Tầm quan trọng:** Giúp đảm bảo hệ thống vận hành ổn định, tối ưu và hạn chế tối đa nguy cơ sự cố gián đoạn (downtime) hoặc độ trễ dữ liệu trong môi trường sản xuất (production).
- **Mục tiêu chính:** Kịp thời phát hiện và xử lý sự cố, tối ưu hóa hiệu năng cụm và đáp ứng đúng các chỉ tiêu cam kết SLOs cũng như SLAs.

---

## 2. Các Chỉ số Giám sát Cốt lõi (Key Metrics)

- **Consumer Lag (Độ trễ tiêu thụ):** Theo dõi khoảng cách giữa tốc độ tạo dữ liệu của Producer và tốc độ xử lý của Consumer. Độ trễ cao là dấu hiệu cảnh báo sự cố từ phía Consumer hoặc cụm Kafka.
- **Mức độ sử dụng tài nguyên Broker (Broker Resource Utilization):** Theo dõi mức độ tiêu thụ CPU, bộ nhớ (Memory) và đĩa cứng (Disk). Tài nguyên chạm ngưỡng cao cho thấy cụm Kafka cần được nâng cấp hoặc bổ sung tài nguyên.
- **Chỉ số Topic & Phân vùng (Topic & Partition Metrics):** Theo dõi số lượng thông điệp được tạo ra, tiêu thụ và số lượng thông điệp bị trễ trên từng topic/partition để phát hiện điểm nghẽn xử lòng dữ liệu.
- **Chỉ số Apache ZooKeeper (ZooKeeper Metrics):** Với hệ thống sử dụng ZooKeeper, cần theo dõi độ trễ (latency), lỗi kết nối (connection errors) và các yêu cầu đang chờ (outstanding requests) để phát hiện sự cố điều phối cluster.

---

## 3. Giám sát Nâng cao (Advanced Monitoring)

- **Giám sát Nhân bản & Chịu lỗi (Replication & Failover):** Kiểm tra trạng thái đồng bộ của các bản sao (in-sync replicas), phát hiện kịp thời các replica bị mất đồng bộ (out-of-sync) và theo dõi hiệu năng hệ thống khi xảy ra chuyển vùng sự cố (failover).
- **Giám sát Hạ tầng Mạng (Network Monitoring):** Theo dõi độ trễ mạng (network latency), tỷ lệ mất gói tin (packet loss) và dung lượng băng thông sử dụng giữa các broker với nhau và với client.
- **Giám sát Xu hướng Hiệu năng (Performance Patterns):** Theo dõi biến động theo thời gian của CPU, bộ nhớ, dung lượng đĩa và thông lượng thông điệp (throughput) để phát hiện sớm các mô hình bất thường.

---

## 4. Công cụ Giám sát Bên thứ ba (Monitoring Tools)

- **Các công cụ phổ biến:** Prometheus, Grafana, Confluent Control Center, Datadog, New Relic, và Splunk.
- **Tiêu chí lựa chọn:** Cần cân nhắc về tính dễ sử dụng, khả năng mở rộng, khả năng tích hợp với hệ thống hiện tại và chi phí triển khai.

---

## 5. Hướng dẫn Triển khai với Prometheus & Grafana (Tutorials)

- **Triển khai với Prometheus & AlertManager:**
    1. _Cài đặt Kafka Exporter:_ Cài đặt module `kafka_exporter` để xuất các chỉ số từ Kafka sang dạng mà Prometheus có thể thu thập.
    2. _Cấu hình Job Scrape:_ Thêm khai báo cấu hình trong file `prometheus.yml` để định kỳ (ví dụ 10 giây/lần) tự động thu thập dữ liệu từ các broker qua đường dẫn `/metrics`.
    3. _Cấu hình AlertManager:_ Khai báo thông tin kênh nhận thông báo (ví dụ cấu hình SMTP gửi Email) trong file `alertmanager.yml`.
    4. _Tạo quy tắc cảnh báo (Alert Rules):_ Tạo file `rules.yml` chứa các quy tắc (ví dụ: phát cảnh báo `HighCPUUsage` khi mức sử dụng CPU của broker vượt quá 80% liên tục trong 1 phút).
    5. _Khởi chạy dịch vụ:_ Chạy lệnh khởi động dịch vụ Prometheus và AlertManager cùng các file cấu hình tương ứng.
- **Triển khai với Grafana:**
    1. _Ưu điểm của Grafana:_ Giao diện trực quan thân thiện, hệ sinh thái mở rộng lớn, khả năng kết nối linh hoạt và hỗ trợ cảnh báo phong phú.
    2. _Kết nối nguồn dữ liệu:_ Cài đặt Grafana và cấu hình kết nối nguồn dữ liệu (Data Source) trỏ tới Prometheus.
    3. _Tạo Dashboard:_ Xây dựng bảng điều khiển trực quan hóa theo thời gian thực các thông số của Broker, Topic và Consumer Group.
    4. _Cài đặt Cảnh báo:_ Thiết lập các quy tắc cảnh báo dựa trên ngưỡng chỉ số trực tiếp trên giao diện Grafana.

---

## 6. Thực hành Tốt nhất (Best Practices)

- **Xác định tập chỉ số cốt lõi:** Lựa chọn tập chỉ số trọng tâm (message throughput, consumer lag, broker resources) bám sát theo chỉ tiêu SLOs/SLAs và cài đặt cảnh báo tự động qua Email, SMS hoặc Slack.
- **Giám sát Nhật ký (Monitor Kafka Logs):** Thu thập và phân tích log của broker/topic bằng các công cụ như ELK Stack (Elasticsearch, Logstash, Kibana), Confluent Control Center hoặc Datadog để phát hiện lỗi, trễ xử lý và sự cố broker.
- **Rà soát Chỉ số Định kỳ:** Thường xuyên đánh giá các biểu đồ trên dashboard để phát hiện sớm các dấu hiệu bất thường và đưa ra điều chỉnh kịp thời.
- **Lập kế hoạch Dung lượng (Capacity Planning):** Đánh giá định kỳ mức độ tăng trưởng dữ liệu nhằm đảm bảo cụm Kafka luôn đáp ứng đủ tài nguyên vận hành.
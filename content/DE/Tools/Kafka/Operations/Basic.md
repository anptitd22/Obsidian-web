	## 1. Quản lý Topic (Adding & Modifying Topics)

### Tạo Topic mới: 

Sử dụng công cụ `bin/kafka-topics.sh` với các tham số chính:

- `--create`: Yêu cầu tạo topic mới.
- `--partitions`: Xác định số lượng phân vùng (giúp phân tán tải và tăng khả năng xử lý song song của consumer). Độ dài tên topic không được vượt quá 249 ký tự.
- `--replication-factor`: Số bản sao của topic. Khuyến nghị đặt từ **2 đến 3** để đảm bảo hệ thống vẫn hoạt động an toàn khi có 1-2 server bị sự cố.

```bash
$ bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic my_topic_name \
    --partitions 20 --replication-factor 3 --config x=y
```

### Thay đổi Topic:
- **Tăng số lượng partition:** Dùng tùy chọn `--alter` kết hợp với `--partitions`. 

	- Không hỗ trợ giảm số lượng Partition

	- Phá vỡ thứ tự topic theo Key (Key Distribution Changes)
		- Khi tăng partition, logic phân vùng mặc định dựa trên `hash(key) % number_of_partitions` sẽ bị thay đổi.
		- **Hậu quả:** Các thông điệp cùng `key` được gửi sau thời điểm tăng partition có thể bị đẩy sang partition mới, làm phá vỡ cam kết về **thứ tự thông điệp (message ordering)** cho key đó.
		- Kafka **không tự động phân bổ lại** các dữ liệu cũ đã tồn tại trên các partition trước đó.

	- Nguy cơ bỏ lỡ/mất dữ liệu ở Consumer (Potential Data Loss)
		- Các consumer đang chạy với cấu hình `auto.offset.reset=latest` có thể bị **bỏ lỡ các thông điệp** ghi vào partition mới trong khoảng thời gian từ khi partition được tạo cho đến khi consumer phát hiện ra partition đó.
		- **Giải pháp khắc phục:** Khuyến nghị chuyển sang sử dụng `auto.offset.reset=by_duration:<duration>` thay vì `latest` để đảm bảo consumer đọc lại được các bản ghi trong khoảng thời gian trễ phát hiện partition.

	- Độ trễ lan truyền Metadata (Metadata Propagation Delay)
		- Producer và Consumer không nhận biết ngay lập tức các partition mới do phải chờ chu kỳ làm mới metadata (quản lý bởi tham số `metadata.max.age.ms`).
		- **Ảnh hưởng:** Có thể xuất hiện tình trạng phân phối thông điệp không đều hoặc gây lag tạm thời cho consumer trong khoảng thời gian ngắn.

	- Rủi ro hỏng hệ thống khi sửa Internal Topics
		- Tuyệt đối **KHÔNG được tăng partition thủ công** cho các topic nội bộ (internal state topics) của Kafka như `__consumer_offsets`, `__transaction_state`, `__share_group_state`, hoặc `__cluster_metadata`.
		- **Hậu quả:** Làm hỏng logic ánh xạ của coordinator, mất nhất quán trạng thái (state inconsistency), dẫn đến hỏng dữ liệu hoặc lỗi toàn hệ thống.

```bash
$ bin/kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic my_topic_name \
    --partitions 40
```

- **Cấu hình động cho topic:** Sử dụng `bin/kafka-configs.sh` với tùy chọn `--add-config` hoặc `--delete-config`.

```bash
$ bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type topics --entity-name my_topic_name --alter --add-config x=y
```

```bash
$ bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type topics --entity-name my_topic_name --alter --delete-config x
```

- **Xóa topic:** Sử dụng `bin/kafka-topics.sh` với tùy chọn `--delete`.

```bash
$ bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic my_topic_name
```

---

## 2. Tắt Broker an toàn & Cân bằng Leader (Graceful Shutdown & Leadership Balancing)

- **Graceful Shutdown (Tắt broker an toàn):**
    - Bật thuộc tính `controlled.shutdown.enable=true`.
    - **Tác dụng:** Tự động đồng bộ log xuống đĩa cứng (tránh mất thời gian phục hồi checksum khi khởi động lại) và di chuyển quyền **Leader** của các partition sang các replica sống khác trước khi dừng, giúp giảm độ trễ gián đoạn xuống chỉ vài miligiây.

- **Cân bằng Leadership (Preferred Replicas):**
    - Kafka có thể tự động khôi phục quyền leader về cho các _preferred replica_ thông qua thuộc tính `auto.leader.rebalance.enable=true`.
    - Hoặc thực hiện cân bằng thủ công bằng lệnh `bin/kafka-leader-election.sh`.

```bash
$ bin/kafka-leader-election.sh --bootstrap-server localhost:9092 --election-type preferred --all-topic-partitions
```

---

## 3. Phân bổ Replica theo Rack (Rack Awareness)

- **Mục đích:** Phân tán các replica của cùng một partition trên các rack hoặc Availability Zone (AZ) khác nhau nhằm tránh mất dữ liệu khi toàn bộ một rack gặp sự cố.

- **Cấu hình:** Thêm dòng `broker.rack=my-rack-id` vào tệp cấu hình của từng broker.

---

## 4. Quản lý Consumer Group & Share Group

- **Kiểm tra trạng thái Consumer Group:**
    - Sử dụng lệnh `bin/kafka-consumer-groups.sh --describe --group <tên_group>` để xem vị trí đọc hiện tại (**CURRENT-OFFSET**), độ lệch dữ liệu chưa đọc (**LAG**), và thông tin các consumer.
    - Thêm tùy chọn `--members` hoặc `--members --verbose` để liệt kê danh sách thành viên đang active và các partition được phân công.

- **Reset Offset:**
    - Sử dụng tùy chọn `--reset-offsets` kết hợp với phạm vi reset như `--to-earliest`, `--to-latest`, `--to-datetime`, hoặc `--shift-by`.
    - Hỗ trợ các chế độ thực thi: `--dry-run` (xem trước), `--execute` (thực thi thật), và `--export` (xuất thông tin ra file CSV).

- **Share Groups:** Quản lý danh sách, offset và Dead-Letter Queue (DLQ) bằng công cụ `bin/kafka-share-groups.sh`.

---

## 5. Mở rộng Cluster & Di chuyển Partition (Cluster Expansion & Data Migration)

- **Công cụ:** Sử dụng `bin/kafka-reassign-partitions.sh` hoạt động qua 3 bước/chế độ:
    1. `--generate`: Tự động tạo kế hoạch di chuyển partition dựa trên danh sách topic và danh sách broker mục tiêu.
    2. `--execute`: Bắt đầu thực thi kế hoạch di chuyển partition từ file JSON.
    3. `--verify`: Kiểm tra trạng thái tiến độ di chuyển (đã hoàn thành, thất bại hay đang xử lý).

- **Giới hạn băng thông (Throttling):**
    - Dùng tham số `--throttle` khi di chuyển partition để giới hạn băng thông mạng giữa các broker, tránh ảnh hưởng đến các ứng dụng đang chạy.
    - Phải chạy lại lệnh với `--verify` sau khi hoàn tất để xóa giới hạn băng thông.

---

## 6. Ngừng hoạt động Broker hoặc Thư mục Log (Decommissioning)

- **Cordoning:** Sử dụng `bin/kafka-configs.sh` đặt `cordoned.log.dirs` để đánh dấu broker hoặc thư mục log không tiếp nhận partition mới.
- **Chuyển dữ liệu & Tắt node:** Di chuyển tất cả partition trên broker/thư mục đó sang các vị trí khác trong cluster trước khi dừng dịch vụ hoặc hủy đăng ký broker (`unregister`).

---

## 7. Cấu hình Quota (Quản lý hạn mức)

- Đặt hạn mức băng thông và tài nguyên cho client/user thông qua `bin/kafka-configs.sh` với các thông số như `producer_byte_rate`, `consumer_byte_rate`, và `request_percentage`.
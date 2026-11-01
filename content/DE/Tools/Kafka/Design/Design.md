## 1. Động lực cốt lõi (Motivation)

- **Mục tiêu:** Nền tảng thống nhất xử lý luồng dữ liệu thời gian thực cho doanh nghiệp lớn (thông lượng cao, xử lý dữ liệu tồn đọng lớn, độ trễ thấp, phân tán và chịu lỗi).
    
- **Bản chất:** Giống một **"database log"** hơn là hệ thống nhắn tin thông thường.
    

## 2. Thiết kế lưu trữ & Hiệu năng (Persistence & Efficiency)

- **Tận dụng File system & OS Pagecache:** Ghi tuần tự (linear writes) cực nhanh (đến 600MB/s). Tránh in-memory cache của ứng dụng Java để giảm tải Garbage Collection (GC); dữ liệu đẩy thẳng vào pagecache của hệ điều hành.
    
- **Hiệu năng không đổi O(1) (Constant Time):** Dùng tệp log append-only thay vì BTree (O(log N)), giúp hiệu suất không phụ thuộc vào kích thước dữ liệu, cho phép lưu trữ dài hạn (ví dụ: 1 tuần).
    
- **Gom nhóm (Batching):** Gom nhiều tin nhắn thành khối lớn (message sets) để tối ưu mạng và I/O.
    
- **Định dạng nhị phân & Zero-Copy:** Định dạng thống nhất từ Producer đến Consumer; dùng lệnh `sendfile` trên Linux để chuyển dữ liệu thẳng từ pagecache sang socket mạng mà không qua user-space.
    
- **End-to-end Batch Compression:** Nén theo đợt bằng GZIP, Snappy, LZ4, hoặc ZStandard; giữ nguyên trạng thái nén trên broker và chỉ giải nén ở consumer.
    

## 3. The Producer (Nhà sản xuất)

- **Cân bằng tải & Phân vùng:**
    
    - Gửi trực tiếp đến **Broker Leader** của partition mà không qua tầng định tuyến trung gian.
        
    - Tự lấy metadata qua bất kỳ node nào trong cụm.
        
    - **Phân vùng ngữ nghĩa (Semantic Partitioning):** Dùng khóa (key) băm để gom các sự kiện cùng loại (ví dụ: User ID) vào chung một partition nhằm đảm bảo thứ tự xử lý.
        
- **Gửi bất đồng bộ & Gom nhóm (Async & Batching):** Tích lũy dữ liệu trong bộ nhớ theo số lượng tối đa hoặc giới hạn thời gian (latency bound) để tăng thông lượng.
    
- **Tùy chọn Độ bền vững (Acks):**
    
    - `acks=0`: Bất đồng bộ hoàn toàn, không chờ phản hồi (tối ưu độ trễ).
        
    - `acks=1`: Chờ broker Leader nhận tin.
        
    - `acks=all` (hoặc `-1`): Chờ toàn bộ bản sao đồng bộ (ISR) ghi nhận (đảm bảo an toàn dữ liệu cao nhất).
        
- **Đảm bảo giao nhận:** Hỗ trợ **Gửi lũy đẳng (Idempotent Delivery)** để tránh trùng lặp tin nhắn do lỗi mạng và **Giao dịch (Transactions)** ghi nguyên tử nhiều topic/partition từ phiên bản 0.11.0.0.
    

## 4. Consumer & Consumer Group (Người tiêu dùng)

- **Cơ chế Pull (Kéo) thay vì Push (Đẩy):** Consumer chủ động kéo dữ liệu giúp tránh bị quá tải, đồng thời tối ưu hóa việc gom nhóm (batching) và hỗ trợ long-poll để tránh busy-waiting.
    
- **Quản lý Vị trí (Offset):**
    
    - Vị trí lưu trữ cực kỳ nhỏ gọn (một số nguyên duy nhất đại diện cho offset tiếp theo trên partition).
        
    - Cho phép **"Tua lại" (Rewind)** offset về quá khứ để xử lý lại dữ liệu khi cần thiết.
        
- **Thành viên Tĩnh (Static Membership):** Cố định ID của consumer qua `GROUP_INSTANCE_ID_CONFIG` giúp tránh hiện tượng rebalance không cần thiết khi khởi động lại ứng dụng.
    
- **Tải dữ liệu ngoại tuyến:** Hỗ trợ consumer chạy batch để nạp dữ liệu định kỳ vào Hadoop hoặc Data Warehouse an toàn không sợ trùng lặp.

>Khi restart đồng loạt hàng chục consumer cùng lúc, các node có thể khởi động chênh nhau vài giây, gây ra rebalance nhiều lần liên tiếp vậy nên kiểm tra **`group.initial.rebalance.delay.ms`** _(mặc định 3000ms = 3 giây)_

## 5. Message Delivery Semantics (Ngữ nghĩa giao nhận tin)

- **Ba mức độ:** At-most-once (Tối đa 1 lần), At-least-once (Ít nhất 1 lần - Mặc định), Exactly-once (Chính xác 1 lần).
    
- **Góc độ Producer:** Dùng `acks=all`, kết hợp Idempotent Delivery và Transactions để loại bỏ bản ghi trùng lặp khi mất kết nối mạng.
    
- **Góc độ Consumer:** Phụ thuộc vào thứ tự xử lý và thời điểm lưu offset.
    
- **Đạt Exactly-once:** Kết hợp tính năng transaction của Kafka (lưu offset và ghi dữ liệu mới vào một transaction nguyên tử) cùng cấu hình `isolation.level=read_committed`.
    

## 6. Using Transactions (Giao dịch)

- **Tính chất:** Chỉ Producer có tính chất giao dịch (update offset và ghi dữ liệu nguyên tử).
    
- **Nguyên tắc cốt lõi:** Phân bổ độc quyền partition cho consumer, cập nhật nguyên tử, và mô hình 1-1 (1 Producer cho mỗi Consumer).
    
- **Cấu hình:**
    
    - Consumer: `isolation.level=read_committed`, `enable.auto.commit=false`.
        
    - Producer: Cấu hình `transactional.id`.
        
- **Xử lý lỗi:** Chia thành lỗi tự phục hồi (`RetriableException`), lỗi cần hủy giao dịch (`AbortableException`), và lỗi nghiêm trọng. Chiến lược phục hồi đơn giản nhất là hủy và khởi tạo lại Producer/Consumer từ đầu.
    

## 7. The Share Consumer

- Cho phép nhiều consumer trong cùng một **Share Group** cùng đọc chung một partition (vượt giới hạn số lượng partition).
    
- Sử dụng **Record Locks** (khóa bản ghi tạm thời, mặc định 30 giây) và cơ chế **xác nhận riêng lẻ** (Acknowledge, Reject, Release) tương tự các hệ thống message truyền thống (RabbitMQ).
    

## 8. Replication (Sao chép & Chịu lỗi)

- **Mô hình Leader - Follower:** Ghi trực tiếp vào Leader; Follower chủ động kéo (pull) log từ Leader.
    
- **Danh sách bản sao đồng bộ (ISR - In-Sync Replicas):**
    
    - Node sống và không bị tụt hậu quá xa sẽ nằm trong ISR. Tập hợp các replica đang hoạt động bình thường, duy trì kết nối heartbeat và không bị chậm trễ quá thời gian `replica.lag.time.max.ms` so với Leader.
        
    - Kafka dùng **ISR thay vì Luật đa số (Majority Vote)**, giúp giảm số lượng bản sao cần thiết (chỉ cần $f+1$ bản sao để chịu $f$ lỗi), tối ưu dung lượng đĩa và thông lượng.
        
- **Unclean Leader Election:** Cho phép cấu hình ưu tiên tính nhất quán (chờ node ISR sống lại, tránh mất dữ liệu) hay ưu tiên tính khả dụng (chọn ngay node đầu tiên thức dậy, có thể mất dữ liệu).
    
- **Quản lý tập trung:** Broker đóng vai trò **Controller** giúp gom nhóm và điều phối thay đổi leadership nhanh chóng khi có sự cố.

- **Cấu hình Bền vững:** Đảm bảo độ an toàn cao bằng cấu hình `acks=all` từ Producer và thiết lập `min.insync.replicas` trên Topic.
    

## 9. Log Compaction (Tinh gọn nhật ký)

- **Mục đích:** Dành cho dữ liệu thay đổi trạng thái (mutable data, database updates), thay vì xóa theo thời gian/dung lượng, Kafka quét và giữ lại **bản ghi có giá trị mới nhất cho mỗi Key**.

- **Ứng dụng:** Thích hợp cho Event Sourcing, Cập nhật Changelog CSDL, và Khôi phục trạng thái (State Recovery) trong Stream Processing.

- **Cấu trúc Log:** Chia thành **Head** (vùng bẩn, chưa nén, mới ghi) và **Tail** (vùng sạch, đã nén, mỗi Key duy nhất 1 giá trị).
    
- **Đặc điểm quan trọng:**
    
    - **Bảo toàn Offset (Offset Immutability):** Offset của tin nhắn không bao giờ thay đổi kể cả khi các tin cũ bị xóa. Nếu đọc vào offset đã xóa, hệ thống trả về offset hợp lệ gần nhất kế tiếp.
        
    - **Tombstones:** Bản ghi có key và giá trị `null` được coi là dấu mốc xóa, sẽ được dọn dẹp sau khoảng thời gian `delete.retention.ms`.
        
- **Log Cleaner:** Tiến trình nền chạy ngầm không chặn đọc/ghi của client để thực hiện việc quét và nén log.
    

## 10. Quotas (Hạn mức)

- Bảo vệ cụm multi-tenant khỏi các client lạm dụng tài nguyên (gây quá tải / DOS).
    
- **Network Bandwidth Quotas:** Giới hạn tốc độ bytes/giây.
    
- **Request Rate Quotas:** Giới hạn tỷ lệ sử dụng CPU/thread.
    
- Xử lý vi phạm bằng cách giữ im lặng kết nối (mute) hoặc tạo ra độ trễ phản hồi (delay) để tạo áp lực ngược (**back-pressure**) cho client.
## 1. Bản chất & Triết lý Thiết kế (Overview)

- **Không phải Resource Manager:** Kafka Streams là một **thư viện ứng dụng** (Java/Scala library), không phải framework quản lý tài nguyên tập trung (như Spark hay Flink). Ứng dụng chạy ở bất kỳ đâu (máy chủ vật lý, VM, container/K8s).
    
- **Kế thừa Kafka nguyên bản:** Xây dựng trực tiếp trên bộ thư viện Kafka Producer/Consumer tiêu chuẩn để tận dụng khả năng điều phối phân tán (_distributed coordination_), phân vùng dữ liệu và tính chịu lỗi có sẵn.
    

![[Pasted image 20261102002610.png]]

## 2. Mô hình Song song: Stream Partitions & Tasks

![[Pasted image 20261102002552.png]]

- **Ánh xạ 1-1 với Kafka Topics:**
    
    - 1 Stream Partition = 1 Kafka Topic Partition (dòng dữ liệu tuần tự, có thứ tự nghiêm ngặt).
        
    - 1 Data Record = 1 Kafka Message.
        
    - Partition Key quyết định lộ trình định tuyến dữ liệu ở cả Kafka broker và Kafka Streams.
        
- **Stream Task – Đơn vị song song cố định:**
    
    - Topology xử lý được chia nhỏ thành các **Task**.
        
    - Số lượng Task được tạo cố định dựa trên số partition của input topic.
        
    - Mỗi Task được gán một danh sách partition cố định và **không bao giờ thay đổi**.
        
    - Task duy trì buffer riêng cho từng partition được gán và xử lý message theo cơ chế _one-at-a-time_, hoàn toàn độc lập và song song mà không cần can thiệp thủ công.
        
- **Giới hạn song song tối đa (Maximum Parallelism):**
    
    - Bị chặn trên bởi **tổng số partition của input topic**.
        
    - _Ví dụ:_ Topic có 5 partition $\rightarrow$ Tối đa 5 instance ứng dụng chạy song song có ích.
        
    - Nếu chạy nhiều hơn 5 instance, các instance dôi dư (_excess instances_) sẽ rơi vào trạng thái nhàn rỗi (_idle_) và tự động tiếp quản công việc nếu một instance đang chạy gặp sự cố.
        
- **Thuật toán phân bổ cố định:** Sử dụng lớp `StreamsPartitionAssignor` nội bộ để cân bằng giữa phân tải (_load-balancing_) và độ dính trạng thái (_stickiness_). Kafka Streams bỏ qua nếu người dùng cấu hình assignor khác.
    

## 3. Mô hình Luồng (Threading Model) & Khả năng Mở rộng

![[Pasted image 20261102002527.png]]

- **Cấu trúc Threading:**
    
    - Mỗi instance ứng dụng cấu hình được số lượng Stream Thread.
        
    - Một Stream Thread có thể thực thi độc lập **một hoặc nhiều Stream Tasks**.
        
- **Chia sẻ trạng thái (Shared State):**
    
    - **Không chia sẻ trạng thái giữa các thread** $\rightarrow$ Không cần cơ chế điều phối hay khóa liên luồng (_no inter-thread coordination_).
        
- **Mở rộng (Scaling) linh hoạt:**
    
    - Scale-out bằng cách bật thêm instance ứng dụng mới hoặc tăng số Stream Thread trên cùng một node.
        
    - _Từ Kafka 2.8+:_ Hỗ trợ scale-up/scale-down số lượng Stream Thread động khi ứng dụng đang chạy; tự động tạo thread mới thay thế thread bị chết mà không cần restart lại toàn bộ client.
        

## 4. Bộ lưu trữ Trạng thái Cục bộ (Local State Stores)

![[Pasted image 20261102002513.png]]

- **Vai trò:** Dùng để lưu trữ và truy vấn dữ liệu trạng thái phục vụ các phép tính có trạng thái (_stateful operations_) như `join()`, `aggregate()`, hoặc xử lý cửa sổ thời gian (_windowing_).
    
- **Quản lý tự động:** Kafka Streams DSL tự động khởi tạo, quản lý và nhúng State Store trực tiếp vào trong từng Stream Task.
    
- **Truy xuất nội bộ:** Mỗi task có state store cục bộ riêng, truy cập qua API với độ trễ cực thấp (thường lưu trên disk/RocksDB hoặc RAM).
    

## 5. Cơ chế Chịu lỗi & Phục hồi (Fault Tolerance)

- **Chịu lỗi cho Stateless Tasks:** Dựa vào cơ chế Consumer Rebalance tiêu chuẩn; nếu một node chết, task được điều phối khởi động lại trên node còn sống và tiếp tục đọc từ partition tương ứng.
    
- **Chịu lỗi cho Stateful Tasks (Changelog Topics):**
    
    - Với mỗi State Store cục bộ, Kafka Streams duy trì một topic nội bộ chuyên dụng gọi là **replicated changelog topic** trên Kafka broker để ghi nhận mọi thay đổi trạng thái.
        
    - Changelog topic được phân vùng tương ứng với local store và **bật Log Compaction** để tránh phình to kích thước vĩnh viễn.
        
    - Khi task chuyển sang máy mới, nó đọc tua lại (_replay_) changelog topic từ broker để khôi phục hoàn toàn dữ liệu của state store về đúng trạng thái trước khi sập.
        

## 6. Tối ưu hóa Thời gian Phục hồi (Recovery Optimization)

Việc tua lại changelog từ Kafka có thể mất thời gian nếu trạng thái lớn. Kafka Streams cung cấp các kỹ thuật tối ưu sau:

- **Standby Replicas (`num.standby.replicas`):**
    
    - Duy trì các bản sao dự phòng "nóng" của state store trên các instance khác (chúng liên tục tiêu thụ changelog ngầm để bắt kịp trạng thái).
        
    - _Từ Kafka 2.6+:_ Cam kết chỉ phân bổ task bị lỗi cho instance nào đã có bản sao trạng thái bắt kịp hoàn toàn (_fully caught-up_), giúp quá trình chuyển đổi (_task migration_) diễn ra gần như tức thì.
        
- **Cơ chế Nhận biết Hạ tầng / Tủ mạng (Rack Awareness):**
    
    - **Phân bổ Standby khác Rack (`rack.aware.assignment.tags`):** Đảm bảo task dự phòng nằm trên một tủ rack/zone vật lý khác với active task, tránh sập toàn bộ khi cả tủ rack gặp sự cố nguồn/mạng.
        
    - **Tối ưu băng thông mạng (`client.rack` & `broker.rack`):** Cấu hình qua `rack.aware.assignment.strategy` để gán task cho client nằm cùng rack với broker Leader, giảm thiểu lưu lượng mạng liên vùng (_cross-rack traffic_).
        
    - **Quy tắc ưu tiên:** Nếu cấu hình cả hai, `rack.aware.assignment.tags` sẽ được ưu tiên hơn `client.rack` khi phân bổ standby task vì hỗ trợ định nghĩa nhiều khóa thẻ (tag keys) linh hoạt hơn.
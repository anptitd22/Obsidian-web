### 1. Nguyên lý cơ bản (Core Principle)

- **Mục đích:** Consumer ghi nhận offset lớn nhất đã xử lý trên từng partition và thực hiện **commit offset** để có thể tiếp tục đọc dữ liệu sau khi restart/crash mà không bị mất dấu.
    
- **Hình thức commit:** Tự động (_automatic_) hoặc thủ công (_manual_) từ phía client.
    

### 2. Group Coordinator & Định vị Coordinator

- **Group Coordinator:** Là một broker được chỉ định riêng cho từng consumer group để quản lý toàn bộ việc lưu trữ (commit) và truy xuất (fetch) offset của nhóm đó.
    
- **Cơ chế gán:** Phân bổ dựa trên thuật toán băm tên của consumer group (`group.id`).
    
- **Cách Consumer tìm Coordinator:**
    
    1. Consumer gửi `FindCoordinatorRequest` đến **bất kỳ broker nào** trong cluster.
        
    2. Đọc kết quả từ `FindCoordinatorResponse` để lấy thông tin kết nối tới broker làm Coordinator.
        
    3. Nếu Coordinator thay đổi (failover/reassign), consumer sẽ tự động phát hiện lại (_rediscover_).
        

### 3. Quy trình Commit Offset (`OffsetCommitRequest`)

- **Nơi lưu trữ:** Coordinator ghi dữ liệu commit dưới dạng append-only vào topic nội bộ đặc biệt: **`__consumer_offsets`**.
    
- **Đặc tính topic `__consumer_offsets`:**
    
    - Áp dụng chính sách **Log Compaction** định kỳ: Chỉ giữ lại offset commit mới nhất cho từng partition của group, dọn dẹp các bản ghi cũ.
        
- **Điều kiện thành công:** Broker chỉ phản hồi commit thành công khi **toàn bộ bản sao (replicas)** của topic offsets đã nhận được dữ liệu.
    
- **Xử lý lỗi:** Nếu quá trình sao chép vượt quá thời gian chờ (_timeout_), commit thất bại và consumer sẽ thực hiện retry kèm backoff.
    
- **Bộ nhớ đệm:** Coordinator lưu đồng thời các offset vào một **in-memory table (cache)** để phục vụ truy vấn cực nhanh.
    

### 4. Quy trình Fetch Offset (`OffsetFetchRequest`)

- **Trạng thái thông thường:** Coordinator trả về ngay vector offset đã commit gần nhất trực tiếp từ **in-memory cache** (không cần đọc đĩa).
    
- **Trạng thái khởi động lại / Chuyển giao Leader:**
    
    - Xảy ra khi Coordinator vừa bật lại hoặc vừa trở thành Leader của một partition trong topic `__consumer_offsets`.
        
    - Coordinator cần thời gian đọc log từ đĩa để nạp (_load_) dữ liệu vào cache.
        
    - **Mã lỗi tạm thời:** Trả về ngoại lệ `CoordinatorLoadInProgressException`. Consumer cần chờ một khoảng thời gian ngắn (backoff) rồi retry lại request.

>[!note]
> - **Group Coordinator:** Là **Kho lưu trữ (Storage)** — giữ lịch sử, lưu offset bền vững vào topic `__consumer_offsets` và in-memory cache.
>- **Consumer Group:** Là **Bên ra quyết định (Driver)** — tự theo dõi tiến độ xử lý thực tế, chủ động phát lệnh commit và gửi request hỏi lại offset mỗi khi cần phục hồi.


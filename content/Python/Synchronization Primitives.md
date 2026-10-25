### 1. Mutex / Lock (Mutual Exclusion)

- **Ý nghĩa:** Khóa loại trừ lẫn nhau. Chỉ cho phép **duy nhất 1** luồng hoặc task được truy cập vào vùng tranh chấp (Critical Section) tại một thời điểm.
    
- **Cách hoạt động:** Ai đến trước thì giữ khóa (`acquire`), làm việc xong thì mở khóa (`release`). Các bên khác đến sau buộc phải dừng lại chờ.
    

### 2. Semaphore

- **Ý nghĩa:** Phiên bản mở rộng của Lock, hoạt động như một máy đếm lượt cấp phép.
    
- **Cách hoạt động:** Cho phép **tối đa $N$ tác vụ** chạy đồng thời. Rất hay dùng để **giới hạn tải (Rate Limiting)**, ví dụ: chỉ cho phép tối đa 5 coroutine gửi request đến server cùng lúc để tránh bị chặn IP.
    

### 3. Event

- **Ý nghĩa:** Một cái cờ (flag) tín hiệu nhị phân (Bật/Tắt).
    
- **Cách hoạt động:** Một hoặc nhiều task đứng đợi cờ này bật (`wait`). Khi một tác vụ khác giải quyết xong việc gì đó, nó dựng cờ lên (`set`), ngay lập tức toàn bộ các task đang chờ sẽ cùng được kích hoạt chạy tiếp.
    

### 4. Condition

- **Ý nghĩa:** Kết hợp giữa Lock và việc chờ đợi một trạng thái/điều kiện phức tạp thay đổi.
    
- **Cách hoạt động:** Một task giữ khóa, kiểm tra điều kiện, nếu chưa thỏa mãn thì nhả khóa và ngủ chờ (`wait`). Khi task khác làm thay đổi điều kiện, nó thông báo (`notify` / `notify_all`) để đánh thức task kia dậy kiểm tra lại.
    

### 5. Barrier

- **Ý nghĩa:** Một "vạch xuất phát" hoặc điểm hẹn tập kết.
    
- **Cách hoạt động:** Cài đặt một số lượng cố định $K$ tác vụ. Mỗi tác vụ đến vạch sẽ dừng lại đợi. Khi đủ cả $K$ tác vụ cùng đến nơi, cổng chắn mới mở ra cho tất cả cùng chạy tiếp.
## 1. Cấu trúc Thư mục & Định dạng File Log

- **Tổ chức thư mục:** Mỗi partition của một topic tương ứng với một thư mục vật lý riêng biệt: `<topic_name>-<partition_id>` (ví dụ: `my-topic-0`, `my-topic-1`).
    
- **Định dạng Log Entry:**
    
    - Gồm: 4 byte biểu diễn độ dài tin nhắn ($N$) + $N$ byte nội dung message.
        
    - Mỗi message được định danh duy nhất bởi một **logical offset 64-bit** (biểu diễn vị trí bắt đầu theo thứ tự tin nhắn trong toàn bộ partition).
        
- **Quy ước đặt tên file:**
    
    - File được đặt tên bằng offset của message đầu tiên mà nó chứa, lấp đầy đủ 20 chữ số (ví dụ: `00000000000000000000.log`).
        
    - File kế tiếp sẽ bắt đầu từ offset liền sau tin nhắn cuối của file trước (khoảng cách dung lượng tương đương cấu hình file size tối đa $S$).
        
- **Thiết kế Offset làm ID (Thay vì GUID):**
    
    - Không dùng GUID vì cần duy trì bảng ánh xạ ngẫu nhiên (b-tree/index tốn kém, cần đồng bộ đĩa).
        
    - Dùng **atomic counter đơn điệu tăng dần** trên mỗi partition $\rightarrow$ Tối ưu hóa cấu trúc tìm kiếm, thao tác đọc/ghi tuần tự cực nhanh.
        

![[Pasted image 20261101223328.png]]

## 2. Thao tác Ghi (Writes)

- **Cơ chế ghi:** Chỉ ghi nối tiếp tuần tự (**serial appends**) vào cuối file log hiện tại (file mới nhất / active segment).
    
- **Roll-over segment:** Khi file log đạt dung lượng giới hạn cấu hình (ví dụ: 1GB), hệ thống tự đóng file cũ và tạo một segment file mới.
    
- **Cơ chế Flush (Độ bền vững - Durability):**
    
    - Cấu hình theo 2 tham số:
        
        - $M$: Số lượng tin nhắn được ghi trước khi ép hệ điều hành flush xuống đĩa cứng.
            
        - $S$: Số giây tối đa trước khi ép flush.
            
    - Đảm bảo: Khi sập hệ thống (crash), rủi ro mất tối đa không quá $M$ tin nhắn hoặc $S$ giây dữ liệu.
        

## 3. Thao tác Đọc (Reads)

- **Đầu vào truy vấn:**
    
    - Offset logic 64-bit của tin nhắn cần đọc.
        
    - Kích thước buffer tối đa ($S$ bytes).
        
- **Quy trình tìm kiếm & đọc:**
    
    1. Dùng biến thể **Binary Search** trên bộ nhớ (in-memory range của các file) để xác định file segment chứa offset cần tìm.
        
    2. Quy đổi offset logic thành vị trí byte offset vật lý tương ứng trong file.
        
    3. Đọc dữ liệu từ file offset đó vào buffer.
        
- **Xử lý kích thước buffer:**
    
    - Buffer trả về một iterator qua các tin nhắn.
        
    - Nếu gặp message ngoại cỡ, client sẽ thử lại và gấp đôi kích thước buffer cho đến khi đọc trọn vẹn.
        
    - Phía cuối buffer nếu bị cắt ngang một message dở dang, hệ thống tự phát hiện nhờ trường 4-byte size delimiting ở đầu entry.
        
- **Đọc dữ liệu mới nhất & Ngoại lệ:**
    
    - Cho phép đọc ngay tin nhắn mới nhất để consumer nhận dữ liệu kiểu "ngay lúc này" (_right now_).
        
    - Nếu client yêu cầu một offset không tồn tại (do đã bị xóa theo retention), broker trả về lỗi `OutOfRangeException` để client chủ động xử lý hoặc reset offset.
        
- **Cấu trúc phản hồi mạng:**
    
    - **Fetch đơn lẻ (`MessageSetSend`):** `total length (4 bytes)` + `error code (2 bytes)` + `danh sách messages (x bytes...)`.
        
    - **Fetch gom cụm (`MultiMessageSetSend`):** `total length (4 bytes)` + `error code (2 bytes)` + `danh sách các MessageSetSend...`.
        

```text
MessageSetSend (fetch result)

total length     : 4 bytes
error code       : 2 bytes
message 1        : x bytes
...
message n        : x bytes


MultiMessageSetSend (multiFetch result)

total length       : 4 bytes
error code         : 2 bytes
messageSetSend 1
...
messageSetSend n
```

## 4. Cơ chế Xóa Dữ liệu (Deletes / Retention)

- **Đơn vị xóa:** Xóa theo từng **file log segment hoàn chỉnh**, không xóa lẻ từng tin nhắn.
    
- **Chính sách xem xét xóa:**
    
    - **Theo Thời gian (Time-based - Mặc định):** Dựa vào timestamp lớn nhất trong file segment để xác định tuổi thọ của cả segment đó so với thời gian lưu trữ quy định.
        
    - **Theo Dung lượng (Size-based):** Mặc định tắt; nếu bật, hệ thống sẽ xóa dần các file segment cũ nhất cho đến khi tổng dung lượng partition nằm trong ngưỡng cho phép.
        
    - Nếu bật cả hai, segment thỏa mãn bất kỳ điều kiện nào cũng sẽ bị xóa.
        
- **Đọc không bị chặn (Lock-free Reads):**
    
    - Danh sách segment được quản lý theo mô hình **Copy-on-Write**.
        
    - Cho phép các tiến trình đọc thực hiện Binary Search trên một snapshot tĩnh bất biến, hoàn toàn không bị khóa (lock) trong lúc tiến trình xóa segment đang diễn ra ở nền.
        

## 5. Đảm bảo Tính Toàn vẹn & Phục hồi khi Sự cố (Guarantees & Recovery)

- **Khởi động & Phục hồi (Log Recovery):**
    
    - Khi khởi động lại sau crash, Kafka duyệt qua toàn bộ tin nhắn trong **segment mới nhất** để kiểm tra tính hợp lệ.
        
    - Điều kiện message hợp lệ:
        
        1. $\text{Kích thước} + \text{Vị trí offset} \le \text{Độ dài thực tế của file}$.
            
        2. Mã **CRC32** tính toán từ payload khớp với mã CRC lưu kèm trong message.
            
    - Nếu phát hiện lỗi/hỏng, log sẽ bị **truncate** (cắt bớt) lùi về offset hợp lệ cuối cùng.
        
- **Xử lý 2 loại lỗi do hệ điều hành (OS Behavior):**
    
    - _Mất block chưa ghi:_ Dữ liệu chưa kịp flush xuống đĩa bị mất do crash.
        
    - _Thêm dữ liệu rác (Nonsense block):_ Hệ điều hành cập nhật inode (tăng kích thước file) trước khi các block dữ liệu thực tế được ghi xuống đĩa; nếu crash xảy ra ở giữa, file có thể bị dính block rác.
        
    - **Cơ chế CRC32** giúp phát hiện chính xác trường hợp này để loại bỏ dữ liệu rác, ngăn chặn nguy cơ làm hỏng toàn bộ chuỗi log.
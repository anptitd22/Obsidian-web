
![[Pasted image 20261027230008.png]]

### 1. Tính chính xác (Accuracy)

- **Ý nghĩa:** Dữ liệu phải phản ánh đúng thực tế khách quan của đối tượng hoặc sự kiện trong thế giới thực. Dữ liệu có thể đúng cú pháp nhưng vẫn sai sự thật.
    
- **Ví dụ:**
    
    - _Hợp lệ nhưng không chính xác:_ Khách hàng tên là Nguyễn Văn An, sinh năm 2000, nhưng trong cơ sở dữ liệu lại ghi năm sinh là 1900. Hệ thống vẫn hiểu đây là ngày hợp lệ, nhưng giá trị sai lệch hoàn toàn so với thực tế.
        
    - _Sai lệch số liệu:_ Đơn hàng phát sinh thực tế có giá trị 500.000 VNĐ, nhưng pipeline ghi nhận nhầm thành 5.000.000 VNĐ do lỗi nhân tỷ giá.
        

### 2. Tính đầy đủ (Completeness)

- **Ý nghĩa:** Mọi dữ liệu bắt buộc cần thiết cho mục đích sử dụng đều phải có mặt, không bị thiếu hụt bản ghi (missing rows) hoặc thiếu thuộc tính (null/empty values) tại các trường trọng yếu.
    
- **Ví dụ:**
    
    - _Thiếu thuộc tính:_ Bảng thông tin người dùng yêu cầu trường `email` hoặc `phone_number` để định danh và gửi thông báo, nhưng 30% bản ghi bị để trống (`NULL`).
        
    - _Thiếu bản ghi:_ Một ngày hệ thống thanh toán xử lý 10.000 giao dịch, nhưng do pipeline CDC bị ngắt quãng, bảng Fact ở Data Warehouse chỉ sync được 8.500 giao dịch (mất 15% dữ liệu trong ngày).
        

### 3. Tính nhất quán (Consistency)

- **Ý nghĩa:** Dữ liệu không mâu thuẫn giữa các bảng, hệ thống, hoặc báo cáo khác nhau khi cùng mô tả một sự vật, hiện tượng.
    
- **Ví dụ:**
    
    - _Mâu thuẫn giữa các hệ thống:_ Trên hệ thống CRM ghi nhận trạng thái khách hàng là `VIP`, nhưng trên hệ thống Billing (thanh toán) lại ghi là `Standard`.
        
    - _Bất đồng bộ trong cùng một bản ghi:_ Một dòng dữ liệu ghi `Trạng_thái_đơn_hàng = "Đã hủy"`, nhưng trường `Thời_gian_giao_hàng_thành_công` lại có timestamp cụ thể.
        

### 4. Tính kịp thời (Timeliness)

- **Ý nghĩa:** Dữ liệu phải sẵn sàng cho người dùng hoặc hệ thống tiêu thụ vào đúng thời điểm nó có giá trị sử dụng. Dữ liệu dù chính xác 100% nhưng đến trễ thì không còn ý nghĩa ra quyết định.
    
- **Ví dụ:**
    
    - _Hệ thống cảnh báo gian lận (Fraud Detection):_ Giao dịch quẹt thẻ ngân hàng bị nghi ngờ lừa đảo, nhưng pipeline xử lý mất 6 tiếng mới đưa ra cảnh báo thay vì vài mili-giây, dẫn đến tiền đã bị rút khỏi tài khoản.
        
    - _Báo cáo kinh doanh:_ Báo cáo doanh thu ngày hôm trước cần hiển thị trước 8:00 sáng cho ban giám đốc, nhưng do pipeline ETL quá tải nên đến tận 14:00 chiều mới hoàn tất.
        

### 5. Tính hợp lệ (Validity)

- **Ý nghĩa:** Dữ liệu phải tuân thủ đúng định dạng, kiểu dữ liệu, độ dài, hoặc tập giá trị cho phép theo quy chuẩn kỹ thuật và nghiệp vụ đã định trước.
    
- **Ví dụ:**
    
    - _Sai định dạng:_ Cột `email` chứa chuỗi `nguyenvanan.gmail.com` (thiếu ký tự `@`), hoặc trường ngày tháng có giá trị `31/02/2026`.
        
    - _Vi phạm miền giá trị:_ Cột `gioi_tinh` chỉ chấp nhận các giá trị enum `['NAM', 'NU', 'KHAC']`, nhưng người dùng nhập vào là `UNKNOWN` hoặc số `123`.
        

### 6. Tính duy nhất (Uniqueness)

- **Ý nghĩa:** Mỗi thực thể trong thế giới thực chỉ được tồn tại một lần duy nhất trong tập dữ liệu, không bị trùng lặp bản ghi (no duplicates).
    
- **Ví dụ:**
    
    - _Trùng lặp kỹ thuật:_ Do producer gửi lại tin nhắn (retry mechanism mà không có idempotency), bảng log ghi nhận 2 dòng có cùng hoàn toàn `event_id`, `timestamp` và `payload`.
        
    - _Trùng lặp nghiệp vụ:_ Một khách hàng đăng ký 2 tài khoản khác nhau bằng 2 số điện thoại, nhưng trùng số Căn cước công dân/CMND, dẫn đến hệ thống tạo ra 2 thực thể khách hàng độc lập làm sai lệch thống kê tổng số user thực.
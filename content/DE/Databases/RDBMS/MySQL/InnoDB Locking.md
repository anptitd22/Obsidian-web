## 1. Khái niệm về các loại khóa (Lock Types)

InnoDB sử dụng nhiều loại khóa khác nhau để kiểm soát quyền truy cập đồng thời, được chia thành các khái niệm chính sau:

- **Shared Locks - S và Exclusive Locks - X:** Đây là hai loại khóa cấp hàng (row-level locking) tiêu chuẩn.
    - **Khóa S** cho phép một giao dịch được đọc một hàng. Nhiều giao dịch có thể cùng giữ khóa S trên một hàng.
    - **Khóa X** cho phép giao dịch thực hiện cập nhật hoặc xóa hàng. Nếu một giao dịch đang giữ khóa X, không có giao dịch nào khác được phép lấy khóa S hay X trên hàng đó, mà bắt buộc phải chờ.
- **Intention Locks - IS, IX:** Đây là loại **table-level locks** giúp InnoDB hỗ trợ khóa đa ranh giới (vừa khóa hàng, vừa khóa bảng). Khóa ý định báo hiệu rằng một giao dịch _dự định_ sẽ thiết lập khóa S (dùng IS) hoặc khóa X (dùng IX) cho các hàng bên trong bảng đó. Mục đích chính của chúng là để ngăn chặn các yêu cầu khóa toàn bộ bảng (như `LOCK TABLES ... WRITE`) mà không cần phải quét qua từng hàng để kiểm tra.

| |`X`|`IX`|`S`|`IS`|
|---|---|---|---|---|
|`X`|Conflict|Conflict|Conflict|Conflict|
|`IX`|Conflict|Compatible|Conflict|Compatible|
|`S`|Conflict|Conflict|Compatible|Compatible|
|`IS`|Conflict|Compatible|Compatible|Compatible|

- **Record Locks:** Đây là khóa được đặt trực tiếp lên một **index record**. Lưu ý quan trọng là ngay cả khi bảng của bạn không định nghĩa chỉ mục nào, InnoDB vẫn sẽ tự động tạo một chỉ mục cụm ẩn (hidden clustered index) để thực hiện khóa bản ghi này.
- **Gap Locks:** Khóa này không khóa một bản ghi cụ thể mà **khóa khoảng trống giữa các bản ghi chỉ mục**, hoặc khoảng trống trước bản ghi đầu tiên/sau bản ghi cuối cùng. Mục đích duy nhất của Gap Lock là **ngăn chặn các giao dịch khác chèn (insert)** dữ liệu vào khoảng trống đó.
- **Next-Key Locks:** Là sự kết hợp của **Khóa bản ghi và Khóa khoảng trống nằm ngay trước bản ghi đó**. Theo mặc định, InnoDB sử dụng loại khóa này để ngăn chặn hiện tượng "hàng bóng ma" (phantom rows) khi tìm kiếm hoặc quét bảng.
- **Insert Intention Locks:** Là một dạng khóa khoảng trống đặc biệt do lệnh `INSERT` thiết lập trước khi chèn dữ liệu. Điểm hay của khóa này là nó **cho phép nhiều giao dịch cùng chèn vào một khoảng trống mà không chặn lẫn nhau**, miễn là vị trí chèn thực tế của chúng không trùng nhau.
- **AUTO-INC Locks:** Đây là một khóa cấp bảng đặc biệt dành cho các giao dịch chèn dữ liệu vào bảng có cột `AUTO_INCREMENT`. Nó đảm bảo các giao dịch phải chờ nhau để các hàng chèn vào nhận được giá trị khóa chính liên tiếp.
- **Predicate Locks for Spatial Indexes:** Vì dữ liệu đa chiều (không gian) không có khái niệm thứ tự tuyệt đối nên không thể xác định "khóa tiếp theo" để dùng Next-Key locks. Thay vào đó, InnoDB dùng khóa vị từ đặt trên Hình chữ nhật giới hạn tối thiểu (MBR) của truy vấn để chặn các cập nhật/chèn dữ liệu ảnh hưởng đến kết quả tìm kiếm không gian.

**2. Các vấn đề và lưu ý quản trị chi tiết**

Từ các khái niệm trên, có một số vấn đề kỹ thuật chi tiết bạn cần nắm rõ:

- **Tính "thuần túy ức chế" của Gap Locks:** Khác với khóa S và khóa X thông thường, **các khóa khoảng trống không xung đột lẫn nhau**. Nghĩa là giao dịch A có thể giữ khóa S trên một khoảng trống, và giao dịch B cũng có thể giữ khóa X trên chính khoảng trống đó. Chúng có thể tồn tại song song vì mục tiêu chung của chúng chỉ là ngăn một giao dịch thứ ba chèn thêm bản ghi vào khoảng trống đó.
- **Ảnh hưởng của Isolation Levels:** Hệ thống khóa của InnoDB thay đổi tùy thuộc vào mức độ cô lập bạn chọn.
    - Ở mức mặc định (`REPEATABLE READ`), InnoDB sử dụng Khóa Next-Key để ngăn chặn triệt để "hàng bóng ma" (phantom).
    - Tuy nhiên, nếu bạn đổi sang mức `READ COMMITTED`, **Gap Locks** sẽ bị vô hiệu hóa đối với các tìm kiếm và quét chỉ mục (chỉ giữ lại cho mục đích kiểm tra khóa ngoại và kiểm tra trùng lặp).
- **Đánh đổi giữa hiệu suất và tính đồng thời:** Cơ chế khóa luôn là sự cân bằng giữa hiệu suất và số lượng giao dịch diễn ra đồng thời. Ví dụ, với khóa AUTO-INC, bạn có thể tinh chỉnh biến `innodb_autoinc_lock_mode` để lựa chọn ưu tiên giữa việc các số tự tăng có thứ tự hoàn toàn dễ đoán hay là tăng tốc độ xử lý các lệnh chèn đồng thời. Hoặc với khóa khoảng trống, nó là sự đánh đổi hiệu năng đặc trưng giữa các mức cô lập giao dịch khác nhau.

Nguồn: [[MySQL  MySQL 9.7 Reference Manual  17.7.1 InnoDB Locking]]
- [[17.6.1.6 AUTO_INCREMENT Handling in InnoDB]]

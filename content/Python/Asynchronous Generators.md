Bất kỳ hàm nào được định nghĩa với `async def` và có chứa từ khóa `yield` thì đều trở thành một **Async Generator** (thuộc kiểu `typing.AsyncGenerator`).

Kết quả trả ra: `<class 'async_generator'>`

Ta cần đến **Asynchronous Generator** vì nó giải quyết điểm nghẽn mà hai công cụ cơ bản (**Sync Generator** và **Async Function thông thường**) không thể làm được cùng lúc.

|**Cách tiếp cận**|**Ưu điểm**|**Nhược điểm chí mạng**|
|---|---|---|
|**Async function trả về danh sách (`async def -> list`)**|Hỗ trợ `await` tác vụ mạng/I/O.|**Tốn RAM & độ trễ cao:** Phải đợi tải xong _toàn bộ_ 100.000 bản ghi rồi mới trả về. Nếu chương trình xử lý giữa chừng bị lỗi, toàn bộ công sức trước đó biến mất.|
|**Sync Generator (`def + yield`)**|Tiết kiệm RAM: Tạo ra đến đâu xử lý đến đó (lazy evaluation).|**Không hỗ trợ `await`:** Không thể chèn các thao tác I/O bất đồng bộ (như gọi API, đọc file async, sleep) mà không làm nghẽn (block) toàn bộ Event Loop.|
**Asynchronous Generator (`async def + yield`) sinh ra để gộp 2 ưu điểm đó:** Vừa xử lý bất đồng bộ không gây nghẽn luồng, vừa sinh dữ liệu từng phần theo nhu cầu (on-demand) để tiết kiệm bộ nhớ.

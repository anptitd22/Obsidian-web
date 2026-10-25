Global Interpreter Lock

Cơ chế được trình thông dịch [[CPython]] sử dụng để đảm bảo chỉ có một luồng thực thi mã bytecode Python tại một thời điểm. 

Điều này đơn giản hóa việc triển khai CPython bằng cách làm cho các object an toàn một cách ngầm định trước sự truy cập đồng thời do lo ngại về race condition [[Reference Counting]]. 

- Việc khóa toàn bộ trình thông dịch giúp cho trình thông dịch dễ dàng hoạt động đa luồng hơn, nhưng lại làm giảm đi phần lớn khả năng song song hóa mà các máy tính đa bộ xử lý cung cấp.

- Tăng tốc độ đơn luồng

- Đơn giản, thận thiện với người dùng như cách mà triết lí nó mang lại

Có các thư viện như numpy, ... được viết trên ngôn ngữ C/C++ và gọi API C của Python hay các tác vụ I/O bound không bị giới hạn bởi GIL.

Về bản chất multi-processing có thể giúp tăng tốc xử lí song song nhờ tạo ra nhiều tiến trình python (mỗi tiến trình có 1 GIL) nhưng đổi lại tốn rất nhiều tài nguyên thậm chí có thể chậm hơn là xử lí trên 1 tiến trình.



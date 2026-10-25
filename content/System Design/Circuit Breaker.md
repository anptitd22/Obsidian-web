Mô hình Circuit Breaker giống như một công tắc an toàn cho các microservice. Giả sử cửa hàng trực tuyến của bạn phụ thuộc vào một dịch vụ thanh toán. Nếu dịch vụ đó bắt đầu gặp sự cố liên tục, mạch ngắt sẽ "kích hoạt" và tạm dừng các cuộc gọi tiếp theo trong một thời gian, ngăn chặn sự lan truyền lỗi và cho phép dịch vụ có thời gian phục hồi. Hãy cùng hiểu rõ hơn về mô hình 

![[Pasted image 20261026002258.png]]

Circuit Breaker qua ví dụ này:
- Cửa hàng của bạn gửi yêu cầu đến dịch vụ thanh toán để xử lý giao dịch. Mọi thứ diễn ra suôn sẻ.

- Đột nhiên, dịch vụ thanh toán gặp sự cố và thất bại ba lần liên tiếp.

- Cầu dao điện (circuit breaker) bị ngắt và chuyển sang trạng thái "mở". Lúc này, khi cửa hàng của bạn cố gắng liên hệ với dịch vụ thanh toán, nó sẽ ngay lập tức nhận được phản hồi lỗi thay vì thử kết nối lại.

- Sau một khoảng thời gian nhất định, cầu dao sẽ chuyển sang trạng thái "nửa mở". Điều này cho phép một vài yêu cầu kiểm tra để xem dịch vụ thanh toán đã hoạt động trở lại hay chưa.

- Nếu các yêu cầu đó thành công, cầu dao sẽ tự động chuyển về trạng thái "đóng" và mọi thứ trở lại bình thường. Nếu chúng thất bại, cầu dao sẽ duy trì trạng thái mở lâu hơn, cho phép dịch vụ thanh toán có thêm thời gian để phục hồi.
Dùng char khi:
- Tốc độ nhanh hơn
- Bộ nhớ cấp phát tĩnh (tốn)
- Độ dài ngắn 0-255
Dùng varchar khi:
- Bộ nhớ cấp phát động (tiết kiệm)
	- Do lưu trữ 1-2 byte tiền tố để tính toán độ dài
- Độ dài lớn
	- Trước MySQL 5.0.3: 255 ký tự .
	- Sau MySQL 5.0.3: 65.535 ký tự.

Trên thực tế thì ít dùng char vì dù nó nhanh hơn varchar nhưng không đáng kể và hiện năng các phần cứng cũng rất mạnh trong khi đó char lại thường tốn bộ nhớ hơn varchar

>[!note] Tại sao char lại nhanh hơn varchar
>Vì độ dài char là cố định nên hệ thống có thể dự đoán trước được mà không cần tính toán còn varchar nó phải lưu trữ 1-2 byte tiền tố để tính toán độ dài 

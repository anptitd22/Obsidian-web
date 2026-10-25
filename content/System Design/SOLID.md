# Giới thiệu

SOLID là từ viết tắt của:
- S: Single Responsibility Principle (SRP)
- O: Open-Closed Principle (OCP)
- L: Liskov Substitution Principle (LSP)
- I: Interface Segregation Principle (ISP)
- D: Dependency Inversion Principle (DIP)

## 1. Single Responsibility Principle (SRP)

Nguyên tắc đầu tiên trong SOLID là "Single Responsibility Principle". Theo nguyên tắc này, một lớp chỉ nên chịu trách nhiệm về một nhiệm vụ cụ thể nào đó. Nếu lớp của bạn đang thực hiện nhiều nhiệm vụ, nó nên được tách thành nhiều lớp khác nhau, mỗi lớp chỉ thực hiện một nhiệm vụ.

![alt][SRP_1.png]

Để tuân thủ SRP, chúng ta tách chức năng lưu vào cơ sở dữ liệu ra khỏi lớp Book:

![alt][SRP_2.png]

Chúng ta có một lớp Book có nhiệm vụ quản lý thông tin về sách và một phương thức để in sách. Nếu chúng ta muốn thêm chức năng lưu thông tin sách vào cơ sở dữ liệu, chúng ta không nên thêm phương thức đó vào lớp Book. Thay vào đó, chúng ta nên tạo một lớp mới có trách nhiệm là lưu thông tin sách.

## 2. Open-Closed Principle (OCP)

Nguyên tắc thứ hai, "Open-Closed Principle", cho rằng phần mềm (lớp, module, function, etc.) phải mở cho việc mở rộng, nhưng đóng cho việc sửa đổi. Điều này có nghĩa là một lớp nên được thiết kế sao cho có thể thêm chức năng mới mà không cần phải sửa đổi code hiện tại.

Ví dụ trái với OCP

![alt][OCP_1.png]

Để tuân thủ OCP, chúng ta nên tạo ra một lớp cho mỗi hình dạng:

![alt][OCP_2.png]

Bạn có một lớp Shape và một phương thức draw để vẽ hình. Nếu bạn muốn thêm chức năng vẽ một hình khác, bạn nên tạo một lớp mới kế thừa từ Shape, chứ không nên sửa đổi lớp Shape và phương thức draw hiện tại.

## 3. Liskov Substitution Principle (LSP)

Nguyên tắc thứ ba, "Liskov Substitution Principle", nói rằng lớp con phải có khả năng thay thế hoàn toàn cho lớp cha mà không làm thay đổi tính đúng đắn của chương trình.

Ví dụ vi phạm LSP

![alt][LSP_1.png]

Để tuân thủ LSP, chúng ta nên tạo ra các lớp phù hợp:

![alt][LSP_2.png]

Nếu Bird là một lớp cơ sở và Penguin là lớp dẫn xuất từ Bird, thì mọi nơi bạn sử dụng Bird cũng nên có thể sử dụng Penguin mà không gây ra lỗi hoặc hành vi không mong đợi.

## 4. Interface Segregation Principle (ISP)

Nguyên tắc thứ tư, "Interface Segregation Principle", khẳng định rằng không nên có các interface lớn mà nhiều chức năng không liên quan đến nhau. Thay vào đó, chúng nên được tách thành nhiều interface nhỏ, mỗi interface chỉ tập trung vào một nhóm chức năng cụ thể.

Ví dụ trái với ISP

![alt][ISP_1.png]

Để tuân thủ ISP, chúng ta tách interface “Worker” thành hai interface:

![alt][ISP_2.png]

## 5. Dependency Inversion Principle (DIP)

Cuối cùng, nguyên tắc "Dependency Inversion Principle" khuyến nghị chúng ta nên phụ thuộc vào các abstraction, không phụ thuộc vào concretions. Điều này giúp giảm sự phụ thuộc trực tiếp giữa các module, giúp code linh hoạt hơn.

Ví dụ vi phạm DIP:

![alt][DIP_1.png]

Để tuân thủ DIP, chúng ta cần phụ thuộc vào các abstraction:

![alt][DIP_2.png]

>[!warning]
>Điều quan trọng nhất khi áp dụng SOLID là không cần phải nguyên tắc áp dụng mọi lúc mọi nơi. Đôi khi, việc đơn giản hóa mã nguồn có thể mang lại lợi ích hơn so với việc tuân thủ một cách mù quáng tất cả các nguyên tắc. Nhưng nếu bạn muốn xây dựng một ứng dụng lớn, phức tạp, thì việc hiểu và áp dụng đúng SOLID sẽ là cần thiết.



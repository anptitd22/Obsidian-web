Kiến trúc có trạng thái (Stateful) so với kiến ​​trúc không trạng thái (Stateless) định nghĩa cách hệ thống quản lý dữ liệu phiên của máy khách trong quá trình tương tác. Điều này ảnh hưởng đến khả năng mở rộng, hiệu suất và thiết kế hệ thống.

- ***Stateful:*** Máy chủ lưu trữ dữ liệu phiên giữa nhiều yêu cầu.
- ***Stateless:*** Mỗi yêu cầu là độc lập và không lưu trữ phiên làm việc nào.
- ***Usage:*** Hệ thống không trạng thái (stateless) được ưu tiên sử dụng cho các hệ thống phân tán và có khả năng mở rộng.

![[stateless_and_statefull_1.png]]

## Stateful Architecture

Máy chủ duy trì trạng thái hoặc thông tin phiên của mỗi máy khách. Điều này có nghĩa là máy chủ theo dõi dữ liệu và ngữ cảnh của máy khách trong suốt nhiều tương tác hoặc yêu cầu.

- Thường liên quan đến việc lưu trữ dữ liệu phiên trong bộ nhớ máy chủ, cơ sở dữ liệu hoặc các cơ chế lưu trữ khác.

- Ví dụ bao gồm các ứng dụng web truyền thống sử dụng phiên máy chủ để lưu trữ dữ liệu người dùng hoặc nội dung giỏ hàng.

Ví dụ: Một trang web mua sắm trực tuyến, nơi máy chủ lưu trữ thông tin đăng nhập và giỏ hàng của người dùng. Nếu người dùng thêm sản phẩm vào giỏ hàng, máy chủ sẽ ghi nhớ những sản phẩm đó trong suốt phiên đăng nhập.

## Stateless Architecture

Máy chủ không lưu trữ bất kỳ thông tin phiên làm việc nào của máy khách giữa các yêu cầu. Mỗi yêu cầu từ máy khách được coi là một giao dịch độc lập.

- Để duy trì phiên người dùng, các kiến ​​trúc không trạng thái thường sử dụng các kỹ thuật như JSON Web Tokens (JWT) hoặc cookie phía máy khách để lưu trữ dữ liệu phiên.

- Được thiết kế để có khả năng mở rộng và chịu lỗi tốt hơn vì chúng không yêu cầu tài nguyên máy chủ để duy trì trạng thái của máy khách.

- Ví dụ bao gồm các API RESTful, trong đó mỗi yêu cầu chứa tất cả thông tin cần thiết để máy chủ xử lý độc lập.

Ví dụ: Một API REST cho ứng dụng di động, trong đó mỗi yêu cầu đều bao gồm một mã thông báo xác thực (JWT). Máy chủ xác minh mã thông báo và xử lý yêu cầu mà không lưu trữ thông tin phiên.
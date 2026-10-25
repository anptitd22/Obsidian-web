Theo định lý CAP, chỉ có hai trong ba đặc điểm mong muốn là Consistency(tính nhất quán), Availability(tính khả dụng) và Partition Tolerance(khả năng chịu đựng phân vùng) có thể được chia sẻ hoặc hiện diện trong hệ thống dữ liệu chia sẻ được kết nối mạng hoặc hệ thống phân tán.

![alt text](CAP_1.png)

## Properties of CAP Theorem

Tính chất của ba đặc điểm hệ thống phân tán mà Định lý CAP đề cập đến:

![alt text](CAP_2.png)

### 1. Consistency

Tính nhất quán định nghĩa rằng tất cả máy khách đều thấy cùng một dữ liệu đồng thời, bất kể chúng kết nối đến node nào trong hệ thống phân tán. Về eventual consistency, các đảm bảo có phần lỏng lẻo. Đảm bảo eventual consistency có nghĩa là máy khách cuối cùng sẽ thấy cùng một dữ liệu trên tất cả các node tại một thời điểm nào đó trong tương lai.

![alt text](CAP_3.png)

**Dưới đây là lời giải thích về sơ đồ trên:** 
- Tất cả các node trong hệ thống đều nhìn thấy cùng một dữ liệu tại cùng một thời điểm. Điều này là do các node liên tục giao tiếp với nhau và chia sẻ các bản cập nhật. 

- Bất kỳ thay đổi nào được thực hiện đối với dữ liệu trên một node sẽ được truyền ngay đến tất cả các node khác, đảm bảo rằng mọi người đều có cùng thông tin cập nhật.

### 2. Availability

Tính khả dụng xác định rằng tất cả các node không bị lỗi trong hệ thống phân tán sẽ trả về phản hồi cho tất cả các yêu cầu đọc và ghi trong một khoảng thời gian giới hạn, ngay cả khi một hoặc nhiều node khác ngừng hoạt động.

![alt text](CAP_4.png)

**Dưới đây là giải thích về sơ đồ trên:** 
- Người dùng gửi yêu cầu, ngay cả khi chúng tôi không thấy các thành phần mạng cụ thể. Điều này ngụ ý rằng hệ thống đã sẵn sàng và đang hoạt động. 

- Mọi yêu cầu đều nhận được phản hồi, dù thành công hay không. Đây là một khía cạnh quan trọng của tính khả dụng, vì nó đảm bảo người dùng luôn nhận được phản hồi.

### 3. Partition Tolerance

Khả năng chịu đựng phân vùng (Partition Tolerance) định nghĩa rằng hệ thống vẫn tiếp tục hoạt động bất chấp việc mất mát hoặc lỗi thông điệp tùy ý ở một số phần của hệ thống. Các hệ thống phân tán đảm bảo khả năng chịu đựng phân vùng có thể phục hồi bình thường từ các phân vùng sau khi phân vùng được phục hồi.

![alt text](CAP_5.png)

**Dưới đây là giải thích về sơ đồ trên:** 
- Xử lý lỗi mạng, một nguyên nhân phổ biến gây ra phân vùng. Điều này cho thấy hệ thống được thiết kế để hoạt động ngay cả khi một số phần của mạng không thể truy cập được. 

- Hệ thống có thể thích ứng với việc phân vùng tùy ý, nghĩa là có thể xử lý các lỗi mạng không thể đoán trước mà không bị lỗi hoàn toàn.

## Trade-Offs in the CAP Theorem

Một hệ thống phân tán chỉ có thể đáp ứng được 2 trong 3 điều kiện chính cùng một lúc, chứ không bao giờ đáp ứng được cả ba. Thông thường, bạn phải lựa chọn giữa Consistency hoặc Availability, nhưng tôi xin nói trước, Partition Tolerance thường là yếu tố "bắt buộc phải có".

Trước hết, chúng ta hãy cùng tìm hiểu xem 'network partition' là gì. 

Đó là điều xảy ra khi giao tiếp giữa các node trong hệ thống không được thông suốt. Trong tình huống như vậy, rất khó để nói liệu một node đã hoàn toàn bị loại khỏi cuộc chơi hay đang gặp phải một số vấn đề khác. 

Vì những vấn đề giao tiếp này chắc chắn sẽ xảy ra, nên việc có một hệ thống có thể hoạt động liên tục là vô cùng quan trọng. Không ai muốn toàn bộ hệ thống của mình ngừng hoạt động, phải không?

### Why only 2?

Trong thực tế, bạn có thể đạt được khả năng chịu đựng phân vùng với tính nhất quán hoặc khả năng chịu đựng phân vùng với tính khả dụng, nhưng không thể đạt được cả ba cùng một lúc, đặc biệt là khi có sự cố mạng.

### Example

Giả sử tôi đang xử lý một tình huống trong đó xảy ra sự cố phân vùng mạng và node A không phản hồi hoặc bị trễ. Lúc này, nó không đồng bộ với các node B, C và D, và trong những trường hợp như vậy, nó cần thời gian để bắt kịp và quay lại trạng thái có dữ liệu mới nhất, đúng không?

### Choose Consistency

Vì vậy, nếu tôi ưu tiên tính nhất quán, tôi sẽ không chuyển hướng các yêu cầu đến node A ngay bây giờ (Hệ thống sẽ trả về lỗi hoặc từ chối các yêu cầu dành cho node A). 

Chúng tôi muốn tất cả các node hiển thị cùng một dữ liệu đồng thời, vì đó chính là mục đích của tính nhất quán. Tuy nhiên, điểm bất lợi ở đây là tính khả dụng, node A sẽ không phản hồi yêu cầu của máy khách trong thời gian này. 

- Độ trễ: Đảm bảo tính nhất quán sẽ làm tăng độ trễ vì thao tác ghi phải truyền đến tất cả các node bản sao trước khi thao tác ghi được coi là hoàn tất và điều này làm cho phản hồi chậm hơn. 
    
- Giảm thông lượng: Do phải chờ truyền lệnh ghi và xác nhận từ tất cả các node nên thông lượng chung của hệ thống bị giảm khi tính nhất quán cao hơn. 
    
- Tính khả dụng thấp hơn: Trong trường hợp mạng giữa các node bị lỗi, các hệ thống nhất quán sẽ có tính khả dụng thấp hơn vì một số yêu cầu bị từ chối hoặc hết thời gian chờ để tránh sự không nhất quán. 
    
- Chi phí tăng: Cần thêm tài nguyên và cơ sở hạ tầng để đảm bảo việc truyền bá và phối hợp kịp thời giữa các node làm tăng chi phí.
    

### Choose Availability

Mặt khác, nếu tính khả dụng quan trọng hơn, node A sẽ tiếp tục xử lý các yêu cầu của khách hàng và hiển thị dữ liệu gần đây nhất mà nó có. 

Nhược điểm là dữ liệu này có thể không khớp với dữ liệu trên các node B, C và D, điều này phá vỡ quy tắc nhất quán: 

- Không nhất quán: Với tính khả dụng cao, các lần đọc có thể trả về dữ liệu cũ hoặc không nhất quán nếu bản cập nhật chưa được truyền đến tất cả các node và điều này vi phạm tính nhất quán. 
    
- Ghi xung đột: Nguy cơ ghi xung đột/ghi đè cao hơn vì các thao tác ghi được phép thực hiện không đồng bộ mà không cần phối hợp. Không xử lý lỗi: Tập trung vào việc phản hồi tất cả các yêu cầu để lỗi và ngoại lệ không thể được xử lý đúng cách. 
    
- Độ cũ không giới hạn: Thông thường không có giới hạn về độ cũ của dữ liệu trả về trong một hệ thống khả dụng, đó là lý do tại sao một số lần đọc có thể trả về dữ liệu rất cũ. 
    
- Giải quyết xung đột khó khăn: Giải quyết các xung đột sẽ khó khăn hơn nếu không có sự phối hợp chặt chẽ. 
    
- Giảm tính toàn vẹn của dữ liệu: Sẽ khó khăn hơn khi áp dụng các ràng buộc và duy trì tính toàn vẹn nếu không có hoạt động đọc/ghi nhất quán và dữ liệu có thể bị hỏng.
    
### So, which one’s your match?

Chúng ta có thể phân loại các hệ thống thành ba loại sau:

![alt text](CAP_6.png)

#### 1. Hệ thống CA 

Hệ thống CA đảm bảo tính nhất quán và khả dụng trên tất cả các node. Hệ thống này không thể thực hiện điều này nếu có sự phân vùng giữa bất kỳ hai node nào trong hệ thống và do đó không hỗ trợ khả năng chịu đựng phân vùng. 

#### 2. Hệ thống CP 

Hệ thống CP cung cấp tính nhất quán và khả năng chịu đựng phân vùng với chi phí thấp hơn tính khả dụng. Khi xảy ra phân vùng giữa hai node, hệ thống sẽ tắt node không khả dụng cho đến khi phân vùng được giải quyết. Một số ví dụ về cơ sở dữ liệu này là MongoDB, Redis và HBase. 

#### 3. Hệ thống AP 

Khả năng sẵn sàng và dung sai phân vùng của Hệ thống AP bị ảnh hưởng bởi tính nhất quán. Khi phân vùng xảy ra, tất cả các node vẫn khả dụng, nhưng những node ở đầu bên kia của phân vùng có thể trả về phiên bản dữ liệu cũ hơn các node khác. Ví dụ: CouchDB, Cassandra và Dyanmo DB, v.v.

![alt text](CAP_7.png)

Trong hình trên, 

- Chúng ta có một hệ thống phân tán đơn giản, trong đó S1 và S2 là hai máy chủ. Hai máy chủ này có thể giao tiếp với nhau. Ở đây, Hệ thống có khả năng chịu phân vùng. Chúng ta sẽ chứng minh rằng hệ thống có thể nhất quán hoặc khả dụng. 

- Giả sử có lỗi mạng và S1 và S2 không thể giao tiếp với nhau. Bây giờ, giả sử máy khách thực hiện lệnh ghi vào S1. Sau đó, máy khách gửi lệnh đọc đến S2. 

- Vì S1 và S2 không thể giao tiếp, chúng có cách nhìn khác nhau về dữ liệu. Nếu hệ thống muốn duy trì tính nhất quán, nó phải từ chối yêu cầu và do đó mất khả năng sử dụng. 

- Nếu hệ thống khả dụng, thì hệ thống phải từ bỏ tính nhất quán. Điều này chứng minh Định lý CAP.

## Use Cases of the CAP Theorem in System Design

Ở đây chúng ta sẽ xem cách chúng ta có thể sử dụng toàn bộ hệ thống đánh đổi trong ứng dụng thực tế:

### 1. Banking Transactions (CP System)

Phát biểu vấn đề:

- Hãy tưởng tượng một giao dịch viên ngân hàng đang cập nhật số dư tài khoản của bạn trên một hệ thống máy tính an toàn. Hệ thống này ưu tiên tính nhất quán (C) và dung sai phân vùng (P).
    

Tại sao chúng tôi sử dụng Hệ thống CP?

- Mỗi giao dịch phải được phản ánh chính xác trên tất cả các máy chủ (tính nhất quán) ngay cả khi các nhánh riêng lẻ gặp sự cố mạng (khả năng chịu đựng phân vùng). 
    
- Sự không nhất quán có thể dẫn đến chi tiêu gấp đôi hoặc số dư không chính xác, những tình huống không thể chấp nhận được trong các giao dịch tài chính.
    
- Mặc dù dữ liệu luôn nhất quán, một số người dùng có thể gặp phải sự chậm trễ tạm thời khi có sự cố mạng do yêu cầu đồng bộ hóa nghiêm ngặt hơn.
    

### 2. Social Media Newsfeed (AP System)

Phát biểu vấn đề:

- Hãy tưởng tượng nguồn cấp tin tức của bạn trên một nền tảng mạng xã hội liên tục cập nhật các bài đăng và câu chuyện mới. Hệ thống này ưu tiên tính khả dụng (A) và dung sai phân vùng (P).
    

Tại sao chúng tôi sử dụng Hệ thống AP?

- Người dùng mong muốn truy cập ngay vào nguồn cấp tin tức của họ (tính khả dụng) ngay cả khi một số phần của mạng tạm thời ngừng hoạt động (dung sai phân vùng). Trong bối cảnh này, những sự không nhất quán nhỏ về dữ liệu, chẳng hạn như việc xem bài đăng của bạn bè sớm hơn một chút trên thiết bị này so với thiết bị khác, là có thể chấp nhận được. 
    
- Dữ liệu có thể không hoàn toàn đồng nhất trên tất cả các máy chủ ngay sau khi cập nhật. Người dùng đôi khi có thể thấy các phiên bản tin tức hơi khác nhau trước khi dữ liệu được truyền đi khắp hệ thống.
    

### 3. Online Shopping Cart (Hybrid System CAP System):

Phát biểu vấn đề:

- Hãy tưởng tượng một giỏ hàng trực tuyến, thêm sản phẩm và thanh toán. Hệ thống này có thể sử dụng phương pháp kết hợp để cân bằng các đôi một CAP.
    

Tại sao chúng ta sử dụng Hệ thống AP và CP?  

- Có thể thêm các mục vào giỏ hàng và hỗ trợ phân vùng (AP), cho phép duyệt web không bị gián đoạn ngay cả khi xảy ra lỗi mạng tạm thời. 
    
- Nhưng khi xác nhận đơn hàng và xử lý thanh toán, hệ thống có thể chuyển sang chế độ CP, đảm bảo tính nhất quán trên tất cả các máy chủ trước khi hoàn tất giao dịch. 
    
- Hệ thống cần được thiết kế cẩn thận để chuyển đổi liền mạch giữa chế độ khả dụng và chế độ nhất quán tại đúng thời điểm nhằm xử lý hiệu quả các giai đoạn khác nhau trong hành trình của người dùng.

![alt text](CAP_8.png)

Trích:
[https://blog.devtrovert.com/p/cap-theorem-why-perfect-distributed](https://blog.devtrovert.com/p/cap-theorem-why-perfect-distributed)

[https://www.geeksforgeeks.org/system-design/cap-theorem-in-system-design/](https://www.geeksforgeeks.org/system-design/cap-theorem-in-system-design/)
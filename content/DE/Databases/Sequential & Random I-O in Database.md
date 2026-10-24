![[sequential_and_random_io1.png]]

Một nhận định đơn giản nhưng mang tính đột phá mà tôi có được về hiệu suất truy vấn cơ sở dữ liệu là nhận ra tầm quan trọng cơ bản của số lượng thao tác đọc tuần tự và ngẫu nhiên đối với cách cơ sở dữ liệu lựa chọn kế hoạch truy vấn, bởi vì các thao tác đọc tuần tự rẻ hơn (rất nhiều) so với các thao tác đọc ngẫu nhiên. Sự khác biệt này dễ hiểu hơn nếu bạn nghĩ về ổ đĩa cơ: các thao tác đọc tuần tự chỉ đơn giản là cho phép ổ đĩa quay và truyền dữ liệu, trong khi các thao tác đọc ngẫu nhiên buộc đầu đọc của ổ đĩa phải tìm vị trí mới để tìm thấy những gì bạn cần. Khoảng cách này nhỏ hơn đối với ổ SSD vì chúng không còn các bộ phận chuyển động, nhưng vẫn đủ lớn để ảnh hưởng đến cách các truy vấn hoạt động. ***Theo tôi, việc nhận ra sự khác biệt về hiệu năng này—và hiểu rằng các chỉ mục là các cấu trúc riêng biệt thường liên quan đến các thao tác tìm kiếm—là chìa khóa để hiểu cách các cơ sở dữ liệu suy luận về hiệu năng truy vấn.***

>[!note]
Trong ổ SSD, chúng ta không quá chú trọng đến I/O ngẫu nhiên so với I/O tuần tự như trong ổ HDD, bởi vì sự khác biệt về độ trễ giữa đọc ngẫu nhiên và đọc tuần tự không lớn. Tuy nhiên, vẫn có một số khác biệt do việc tìm nạp trước dữ liệu, đọc các trang liền kề và song song hóa nội bộ. (Alex Petrov, Database Internals: A Deep Dive into How Distributed Data Systems Work, O'Reilly Media, 2019)

Bài học này đến với tôi khoảng mười năm trước. Chúng tôi có một truy vấn báo cáo nặng nề chạy mỗi sáng trên MySQL replica, thường mất khoảng 10 phút để hoàn thành. Tuy nhiên, một ngày nọ, người dùng bắt đầu phàn nàn rằng báo cáo không được thực thi. Chúng tôi bắt đầu điều tra và nhận ra rằng không phải truy vấn không được thực thi, mà thực chất là ***nó vẫn đang chạy. Qua đêm, thời gian truy vấn đã tăng từ 10 phút lên hơn một giờ***. Ban đầu, nghĩ rằng đó chỉ là sự cố tạm thời—có lẽ cơ sở dữ liệu đang chịu tải quá lớn trong thời gian báo cáo—chúng tôi quyết định bỏ qua và xem liệu nó có tự khắc phục vào ngày hôm sau hay không.

Ngày hôm sau đến, và thật ngạc nhiên, truy vấn vẫn chậm một cách khó chịu. Được rồi—đã đến lúc phải tìm hiểu sâu hơn. Điều đầu tiên chúng tôi nhận thấy là cơ sở dữ liệu đang thực hiện quét toàn bộ bảng để thực thi truy vấn, hoàn toàn bỏ qua chỉ mục mà chúng tôi đã thiết lập. Lúc đầu, chúng tôi thậm chí còn không chắc liệu chỉ mục đó đã từng được sử dụng hay chưa—chắc chắn cơ sở dữ liệu không thể tự nhiên thay đổi kế hoạch truy vấn chỉ sau một đêm… hay là không?

Dường như đúng là vậy. Khi chúng tôi chạy truy vấn với tùy chọn FORCE INDEX, thời gian thực thi ngay lập tức giảm xuống còn 10 phút như dự kiến. MySQL quả thực đã thay đổi quyết định qua đêm. Được rồi—việc buộc tạo chỉ mục đã khắc phục được sự cố, nhưng chúng tôi vẫn muốn hiểu tại sao điều đó lại xảy ra. Tại thời điểm đó, với kiến ​​thức hạn chế về cơ sở dữ liệu, chúng tôi có hai câu hỏi lớn trong đầu.

1. If there’s an index, shouldn’t the database always use it?
2. How did MySQL get this so wrong?

### 1. If there’s an index, shouldn’t the database always use it?

Chỉ mục trong cơ sở dữ liệu là các cấu trúc riêng biệt so với chính các hàng trong bảng. Chúng giúp định vị nhanh các hàng khớp với các bộ lọc nhất định, nhưng có một vấn đề: sau khi tìm thấy các hàng đó, cơ sở dữ liệu vẫn phải "nhảy" đến dữ liệu bảng thực tế trên đĩa để truy xuất các cột còn lại. Những lần nhảy này thường liên quan đến thao tác random I/O.

Giả sử bạn có một bảng với 1.000.000 hàng và bạn chạy một truy vấn lọc theo cột được lập chỉ mục mà chỉ khớp với 10 hàng. Trong trường hợp này, cơ sở dữ liệu sẽ sử dụng chỉ mục — việc thực hiện 10 lần tìm kiếm ngẫu nhiên sẽ tiết kiệm chi phí hơn nhiều so với việc quét toàn bộ 1.000.000 hàng theo trình tự. Nhưng nếu bộ lọc của bạn khớp với 900.000 hàng, thì câu chuyện sẽ đảo ngược. ***Cơ sở dữ liệu sẽ bỏ qua chỉ mục và chỉ quét toàn bộ bảng theo trình tự, bởi vì việc đọc tất cả các hàng theo thứ tự sẽ tiết kiệm chi phí hơn nhiều so với việc nhảy lung tung để tìm kiếm gần như từng hàng một.***

Quay lại câu chuyện: mỗi sáng, phiên bản MySQL của chúng tôi đều âm thầm tự hỏi mình câu hỏi tương tự: “Sử dụng chỉ mục để tìm tất cả các hàng trong 30 ngày qua rồi truy xuất từng hàng một có tiết kiệm hơn không, hay là nên quét toàn bộ bảng theo thứ tự?” Cơ sở dữ liệu dựa vào ước tính chi phí cho những quyết định này, và những ước tính đó liên tục được cập nhật khi dữ liệu thay đổi. Cho đến ngày định mệnh đó, MySQL tin rằng sử dụng chỉ mục là con đường thông minh hơn. Nhưng rồi, chỉ sau một đêm, nó quyết định rằng quét toàn bộ bảng là cách tốt nhất.

### 2. How did MySQL get this so wrong?

Lời giải thích mà chúng tôi đưa ra vào thời điểm đó về lý do tại sao MySQL lại đưa ra quyết định tồi tệ như vậy là do ước tính chi phí của nó vẫn dựa trên giả định về ổ đĩa cơ (và công bằng mà nói, SSD không phổ biến như bây giờ), trong khi cơ sở dữ liệu của chúng tôi đang chạy trên SSD. ***Các thao tác random I/O rẻ hơn nhiều so với dự đoán của MySQL, điều này khiến nó tin rằng việc quét toàn bộ bảng sẽ nhanh hơn.***

>[!note]
>Nếu bạn nghĩ rằng MySQL “nên làm tốt hơn”, thì nên đọc tài liệu của PostgreSQL ([documentation](https://www.linkedin.com/redir/redirect?url=https%3A%2F%2Fpostgresqlco%2Enf%2Fdoc%2Fen%2Fparam%2Frandom_page_cost%2F&urlhash=WhAr&trk=article-ssr-frontend-pulse_little-text-block)) về tham số random_page_cost để thấy rằng mọi thứ không đơn giản như vậy. Hãy để ý xem việc tìm ra một giá trị mặc định hợp lý khó khăn như thế nào: giá trị được đặt là 4.0. Theo tài liệu, con số đó đã thấp đối với ổ đĩa cơ nhưng lại quá cao đối với ổ đĩa trạng thái rắn (SSD) (trong trường hợp này, con số lý tưởng nên gần với 1.1 hơn). Vậy tại sao lại là 4.0? Tài liệu giải thích: “Giá trị mặc định có thể được hiểu là mô phỏng truy cập ngẫu nhiên chậm hơn 40 lần so với truy cập tuần tự, trong khi giả định 90% số lần đọc ngẫu nhiên sẽ được lưu vào bộ nhớ đệm.”

### Some Pictures Worth a Thousand Words

Hãy tưởng tượng bạn có một bảng với 12 hàng, mỗi hàng có một cột ngày tháng, và mục tiêu của bạn là tìm tất cả các hàng từ ngày 1 tháng 7 đến ngày 5 tháng 7.

Nếu bạn không có chỉ mục (hoặc nếu cơ sở dữ liệu quyết định không sử dụng chỉ mục, như chúng ta vừa thấy), điều này sẽ xảy ra: cơ sở dữ liệu sẽ quét qua tất cả các hàng trong bảng, lọc ra những hàng không khớp với phạm vi ngày. Thật dễ dàng để thấy cách tiếp cận này trở nên có vấn đề như thế nào khi bảng của bạn có hàng triệu hàng và bạn chỉ quan tâm đến một vài hàng.

![[sequential_and_random_io2.png]]

Without an index, the database scans all rows sequentially, checking each one against the filter

Nếu bạn có chỉ mục (và cơ sở dữ liệu quyết định sử dụng nó), đây là những gì xảy ra: cơ sở dữ liệu nhanh chóng định vị tất cả các hàng khớp trong chỉ mục, sau đó nhảy đến từng hàng để lấy dữ liệu thực tế, vì chỉ mục chỉ lưu trữ con trỏ đến các hàng thực tế.

![[sequential_and_random_io3.png]]

Using an index: the database quickly finds matching row IDs, but then needs to jump back and forth to fetch the actual data.

Mười năm trước, tôi nghĩ rằng việc sử dụng bất kỳ chỉ mục nào cũng luôn tốt hơn việc quét toàn bộ bảng — ***bởi vì tôi đã tự hỏi mình câu hỏi sai: “Đọc 4 hàng có nhanh hơn đọc 12 hàng không?”. Như chúng ta vừa thấy, câu hỏi thực sự là: “Đọc 4 hàng một cách ngẫu nhiên có nhanh hơn đọc 12 hàng theo trình tự không?”***. Đó là loại đánh đổi mà các cơ sở dữ liệu liên tục đánh giá khi chúng ước tính chi phí của mỗi cách có thể để thực hiện một truy vấn.

![[sequential_and_random_io4.png]]

Illustrative Cost Estimator: 12 sequential reads or 4 random reads ? Here, it predics the full scan is faster.

Lưu ý: bài viết này bao gồm một số điểm đơn giản hóa (mà tôi dự định sẽ đi sâu hơn trong các bài viết sau) để tránh đi quá sâu vào vấn đề phức tạp.

- Cơ sở dữ liệu hoạt động theo khối/trang, chứ không phải từng hàng riêng lẻ — điều này có một số tác động quan trọng đến việc tính toán chi phí truy vấn.
- Các chỉ mục thực tế được lưu trữ trong các cấu trúc dạng cây (ví dụ: B+Trees), chứ không phải là các sơ đồ dạng bảng đơn giản như trong hình minh họa.
- Các chuyên gia tối ưu hóa chi phí thực tế xem xét nhiều yếu tố khác khi tính toán chi phí kế hoạch, không chỉ số lượng các thao tác I/O tuần tự và ngẫu nhiên.

Nguồn: [[Clippings/DE/Database/Topic/Sequential vs Random IO|Sequential vs Random IO]]


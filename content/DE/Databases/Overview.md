Khái niệm về DBMS và RDBMS hiện nay khá là mỏng manh do RDBMS hiện nay cũng hỗ trợ các tính năng mà DBMS mang hiện như lưu trữ JSON, graph, ... và DBMS hiện nay như MongoDB cũng đã hỗ trợ đầy đủ tính chất ACID cho transaction cũng như có thể cung cấp một schema cứng nhắc như RDBMS.

- MongoDB không có Foreign Keys nên người dùng phải tự đảm bảo Consistency

- Multi-document Transactions trong MongoDB tốn rất nhiều chi phí

Mỗi một kiểu thiết kế sẽ phù hợp cho từng bài toán khác nhau:

- Bài toán về bán hàng, giao dịch, lưu trữ thông tin: cần được thiết kế normalize để tránh dư thừa dữ liệu đổi lại cần tạo nhiều bảng, tốc độ chậm khi sử dụng cần join rất nhiều để lấy đầy đủ thông tin nên RDBMS phù hợp hơn do nó có nhiều thuật toán tối ưu JOIN và Foreign Keys đảm bảo tính nhất quán của dữ liệu.

- Bài toán về trò chuyện, vector: cần tốc độ nhanh, dữ liệu được flatern hạn chế JOIN, schema có cấu trúc phức tạp lồng nhau thì phù hợp với DBMS hơn.
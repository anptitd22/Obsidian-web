## Pessimistic

Hoạt động dựa trên niềm tin rằng xung đột giữa các transaction có thể xảy ra, và do đó nó sử dụng khóa để ngăn chặn xung đột bản ghi trong cơ sở dữ liệu. Một số hậu quả là các loại transaction khác chỉ có thể truy cập bản ghi ở chế độ read-only hoặc phải đợi cho đến khi bản ghi này được mở khóa.

Quy trình các giai đoạn transaction:

```
Validate -> Read -> Compute -> Write
```

Ưu điểm:

- Ngăn ngừa xung đột: Do đó, các bản ghi không thể bị thay đổi bởi các giao dịch khác do cơ chế khóa, vì vậy sẽ có ít xung đột khi transaction chưa hoàn tất.

- Độ tin cậy cao: Phương pháp này hữu ích trong các tình huống có mức độ tranh chấp transaction cao nhờ các khóa đóng vai trò quan trọng trong việc duy trì tính nhất quán (consistency) của dữ liệu.

Nhược điểm:

- Rủi ro tắc nghẽn: Vì vậy, khi làm việc với các khóa, cần phải hiểu rằng chúng có thể dẫn đến tắc nghẽn, cụ thể là deadlock và do đó làm cho việc lập trình trở nên phức tạp hơn.

- Giảm khả năng xử lý đồng thời: Khóa làm giảm khả năng xử lý đồng thời của hệ thống vì các giao dịch có thể phải chờ để có được quyền sử dụng các khóa cần thiết.

- Chi phí lưu trữ cao hơn: Hầu hết các trường hợp, nó đòi hỏi nhiều không gian lưu trữ hơn vì cần phải duy trì các khóa hoặc các cơ chế `concurrency control` khác.

Khóa thực thi: 

- For update: khóa row không cho transaction khác đọc, update cho đến khi transaction đó commit hoặc roll_back

- For share: khóa row cho các transaction khác đọc nhưng không thể update cho đến khi transaction đó commit hoặc roll_back

Ứng dụng:

- Các transaction liên quan đến tiền, tránh race condition

## Optimistic

Dựa trên giả định rằng xung đột transaction rất hiếm khi xảy ra. Thay vì sử dụng khóa trong quá trình thực thi thông thường, nó cho phép nhiều transaction diễn ra đồng thời và chỉ kiểm tra xung đột tại thời điểm commit. Nếu phát hiện xung đột trong quá trình xác thực, transaction sẽ bị hủy bỏ.

Quy trình các giai đoạn transaction:

```
Read -> Compute -> Validate ->  Write
```

Ưu Điểm

- **Hoàn tác (Rollback) đơn giản:** Quá trình hủy bỏ giao dịch diễn ra rất dễ dàng khi xảy ra xung đột.
    
- **Không bị hoàn tác dây chuyền (No Cascading Rollbacks):** Giao dịch thao tác trực tiếp trên bản sao dữ liệu cục bộ (local copy/private workspace) thay vì ghi thẳng vào cơ sở dữ liệu chính trước khi kiểm tra hợp lệ.
    

Nhược Điểm

- **Chi phí cao khi xung đột nhiều:** Nếu tỉ lệ tranh chấp dữ liệu lớn, việc phải rollback và thử lại (retry) liên tục làm tiêu tốn rất nhiều tài nguyên hệ thống.
    
- **Bất lợi cho các transaction lớn (Starvation/Repeated Rollback):** Khi xảy ra xung đột giữa transaction lớn (thao tác nhiều bản ghi) và transaction nhỏ, transaction lớn thường xuyên bị ép rollback lặp đi lặp lại vì phạm vi xung đột của nó rộng hơn.

## Bảng So Sánh

| **Tiêu chí**                       | ** Pessimistic**                                                                                    | **Optimistic**                                                                                                                                         |
| ---------------------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Cơ chế khóa**                    | **Khóa bản ghi (Locking):** Ngăn chặn người khác thay đổi dữ liệu trong lúc giao dịch đang đọc/sửa. | **Không khóa bản ghi (No lock):** Cho phép sửa tự do, chỉ kiểm tra xem dữ liệu có bị thay đổi giữa lúc đọc (`SELECT`) và lúc nộp (`SUBMIT`) hay không. |
| **Tần suất xung đột thực tế**      | Áp dụng khi giả định **xung đột xảy ra thường xuyên** giữa các transaction.                         | Áp dụng khi giả định **xung đột rất hiếm khi xảy ra**.                                                                                                 |
| **Thời điểm đồng bộ hóa**          | Thực hiện ngay ở **giai đoạn đầu** của vòng đời thực thi (lấy khóa trước khi làm việc).             | Thực hiện ở **giai đoạn sau cùng** (trước khi ghi đĩa) hoặc trì hoãn đến lúc commit.                                                                   |
| **Độ phức tạp thiết kế**           | Thiết kế và lập trình đơn giản hơn.                                                                 | Phức tạp hơn trong thiết kế và xử lý logic giải quyết xung đột/retry.                                                                                  |
| **Chi phí lưu trữ bộ nhớ**         | Tốn chi phí lưu trữ cao hơn (phải duy trì Lock Table trong bộ nhớ).                                 | Tiết kiệm chi phí lưu trữ hơn do không cần duy trì bảng khóa phức tạp.                                                                                 |
| **Mức độ đồng thời (Concurrency)** | Thấp hơn (do các giao dịch phải chờ khóa - Lock Wait).                                              | Cao hơn rất nhiều (nhiều tiến trình có thể đọc/sửa cùng lúc).                                                                                          |
| **Môi trường phù hợp**             | Nơi có **nhiều tranh chấp dữ liệu** (tỉ lệ ghi đồng thời cao).                                      | Nơi có **ít hoặc hiếm khi xảy ra tranh chấp dữ liệu** (hệ thống đọc nhiều, ghi ít).                                                                    |
| **Luồng các giai đoạn (Workflow)** | _(Validate $\rightarrow$ Read $\rightarrow$ Compute $\rightarrow$ Write)_                           | _(Read $\rightarrow$ Compute $\rightarrow$ Validate $\rightarrow$ Write)_                                                                              |
| **Cách tiếp cận xung đột**         | Chủ động **ngăn ngừa** xung đột xảy ra ngay từ đầu.                                                 | **Cho phép xung đột xảy ra**, sau đó mới phát hiện và xử lý.                                                                                           |
| **Quy mô phù hợp**                 | Phù hợp cho database nhỏ hoặc bảng có ít bản ghi (nơi dễ tranh chấp vào cùng một vùng).             | Phù hợp cho database lớn hoặc bảng có nhiều bản ghi (xác suất chạm chung một dòng là rất thấp).                                                        |


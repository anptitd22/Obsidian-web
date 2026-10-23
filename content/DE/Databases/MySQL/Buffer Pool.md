---
title: Buffer Pool
tags:
  - MySQL
---
## Buffer Pool

Là một vùng nhớ đệm nơi InnoDB sẽ truy cập trực tiếp để lấy dữ liệu được sử dụng thời xuyên trước khi xuống ổ cứng để đọc giúp tăng tốc độ đọc dữ liệu. Trên các máy chủ chuyên dụng, thường có đến 80% bộ nhớ vật lý được dành cho vùng nhớ đệm.

Thuật toán quản lí là LRU (Least Recently Used). Khi cần thêm chỗ trống để thêm một trang mới vào vùng nhớ đệm, trang được sử dụng ít nhất gần đây sẽ bị loại bỏ và một trang mới được thêm vào giữa danh sách. Chiến lược chèn điểm giữa này coi danh sách như hai danh sách con.

- Ở đầu trang là danh sách phụ các trang mới ("mới") được truy cập gần đây.

- Ở cuối trang là danh sách con các trang cũ được truy cập ít gần đây hơn.

![[Pasted image 20261024001826.png]]

Thuật toán giữ lại các trang được sử dụng thường xuyên trong danh sách con mới. Danh sách con cũ chứa các trang được sử dụng ít thường xuyên hơn; những trang này là đối tượng có thể bị loại bỏ.

## Change Buffer

Là cấu trúc dữ liệu lưu trữ tạm các thay đổi (INSERT, UPDATE, DELETE) đối với các trang **Secondary Index** khi trang đó **chưa nằm trong Buffer Pool**.

![[Pasted image 20261024020545.png]]

**Mục đích duy nhất:** **Loại bỏ Random Disk I/O**.

- Cập nhật Secondary Index thường nằm rải rác trên đĩa. Thay vì phải đọc ngay từng trang ngẫu nhiên từ đĩa lên RAM để sửa, InnoDB lưu tạm thao tác vào Change Buffer.

**Vị trí lưu trữ:**

- **Trong RAM:** Chiếm một phần bộ nhớ của Buffer Pool.
    
- **Trên đĩa:** Nằm trong System Tablespace (được lưu bền vững khi server tắt).

**Chỉ áp dụng cho Secondary Index:** Không dùng cho Clustered Index (Primary Key).
    
**Không hỗ trợ:**

- Secondary Index có chứa cột giảm dần (**Descending Index**).
        
- Bảng có Primary Key chứa cột giảm dần.
        
- Secondary Index duy nhất (**UNIQUE**): Bắt buộc phải đọc đĩa ngay để kiểm tra trùng lặp (trừ khi đã bật `unique_checks=0`).

## Doublewrite Buffer

**Doublewrite Buffer (DWB)** dùng để **chống rách trang dữ liệu (partial/torn page)** khi sập nguồn hoặc crash đột ngột.

- **Vấn đề:** MySQL ghi từng trang **16 KB**, nhưng ổ cứng ghi từng khối nhỏ **4 KB**. Đang ghi dở mà mất điện thì trang dữ liệu bị hỏng (checksum sai) — lúc này **Redo Log bó tay** vì không thể áp bản vá lên một trang đã nát cấu trúc.
    
- **Cách giải quyết:** Trước khi ghi vào file dữ liệu chính, InnoDB ghi nguyên trang 16 KB vào vùng đệm **Doublewrite Buffer** trước.
    
- **Khi crash:** Nếu trang ở file chính bị rách, InnoDB lấy bản sao nguyên vẹn trong DWB đè lại, sau đó mới dùng Redo Log để khôi phục tiếp dữ liệu mới nhất.

Liên quan:
- [[Glossary]]
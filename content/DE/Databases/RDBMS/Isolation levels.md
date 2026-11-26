---
title: Isolation levels
tags:
  - MySQL
  - PostgreSQL
  - RDBMS
---
## Dirty Read

> **Tóm gọn:** Đọc trộm dữ liệu chưa chốt, người ta hủy thì mình "hớ".

- **T1:** Đang sửa đổi dữ liệu trên hàng A nhưng **chưa commit**.
    
- **T2:** Nhảy vào đọc hàng A $\rightarrow$ lấy đúng dữ liệu T1 đang sửa dở.
    
- **T1:** Gặp lỗi và bấm **Rollback** (dữ liệu phục hồi như cũ).
    
- **Hậu quả:** T2 đã đọc và mang đi xử lý một mẩu dữ liệu "ảo" chưa từng tồn tại chính thức.

## Non-repeatable Read

> **Tóm gọn:** Cùng một hàng, đọc 2 lần trong 1 transaction ra 2 giá trị khác nhau vì người khác đã sửa và commit.

- **T1:** Đọc hàng A, thấy giá trị là $100$.
    
- **T2:** Sửa hàng A thành $200$ (hoặc xóa luôn hàng A) và **Commit thành công**.
    
- **T1:** Đọc lại đúng hàng A đó lần nữa $\rightarrow$ thấy giá trị biến thành $200$.
    
- **Điểm cốt lõi:** Lỗi này xảy ra trên **cùng một dòng dữ liệu cụ thể** bị **UPDATE/DELETE**.

## Phantom

> **Tóm gọn:** Cùng một câu query tìm kiếm, lần 2 xuất hiện thêm "bóng ma" mới chui vào danh sách vì người khác vừa thêm dòng mới.

- **T1:** Đếm hoặc truy vấn theo điều kiện: _"Lấy tất cả học sinh có điểm > 8"_ $\rightarrow$ trả về **5 dòng**.
    
- **T2:** Chèn thêm một học sinh mới điểm 9 vào bảng (`INSERT`) và **Commit thành công**.
    
- **T1:** Chạy lại đúng câu query đó lần nữa $\rightarrow$ bỗng nhiên trả về **6 dòng**.
    
- **Điểm cốt lõi:** Lỗi này không phải do sửa dòng cũ, mà do xuất hiện thêm **dòng mới (`INSERT`)** thỏa mãn tập điều kiện tìm kiếm.

## Phương pháp

Có 4 isolation level chính và tùy vào các database thì nó có chức năng khác nhau:

- Uncommitted Read

- Committed Read

- Repeatable Read

- Serializable

Ngoài ra còn có:

- Snapshot
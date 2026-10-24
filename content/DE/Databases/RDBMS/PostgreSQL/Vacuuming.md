---
title: VACUUMMING
tags:
  - PostgreSQL
  - RDBMS
---
## 4 Mục Tiêu Cốt Lõi của Lệnh VACUUM

1. **Thu hồi / Tái sử dụng không gian đĩa (Disk Space Reclamation)**: Dọn các bản ghi chết (_dead tuples_) sinh ra từ `UPDATE` / `DELETE` (do cơ chế MVCC giữ phiên bản cũ).
    
2. **Cập nhật số liệu thống kê (Update Planner Statistics)**: Thu thập dữ liệu phân bố (phân tích bởi `ANALYZE`) cho Query Planner tạo execution plan chính xác.
    
3. **Cập nhật Visibility Map (VM)**:
    
    - Giúp các lần VACUUM sau bỏ qua các trang toàn bộ tuple đã hiển thị (_all-visible_).
        
    - Tăng tốc **Index-Only Scans** (không cần nhảy vào Heap table để kiểm tra tính hiển thị).
        
4. **Chống tràn số Transaction ID (Prevent XID Wraparound)**: Đóng băng (_freeze_) các tuple quá cũ để tránh mất dữ liệu diện rộng.

## Standard VACUUM vs. VACUUM FULL vs. TRUNCATE

| **Tiêu chí**             | **Standard VACUUM**                                                                                             | **VACUUM FULL**                                                                           | **TRUNCATE**                                                  |
| ------------------------ | --------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Mức khóa (Locking)**   | `SHARE UPDATE EXCLUSIVE` _(chạy song song với SELECT, INSERT, UPDATE, DELETE; chỉ chặn DDL như ALTER TABLE)_    | `ACCESS EXCLUSIVE` _(chặn toàn bộ đọc/ghi trên bảng)_                                     | `ACCESS EXCLUSIVE`                                            |
| **Cơ chế thu hồi đĩa**   | Đánh dấu không gian trống để tái sử dụng nội bộ; **không** trả dung lượng cho OS (trừ trang trống ở cuối bảng). | Viết lại hoàn toàn một file bảng mới nén chặt dữ liệu; **trả lại đĩa cho OS**.            | Xóa trắng dữ liệu ngay lập tức; giải phóng dung lượng cho OS. |
| **Yêu cầu đĩa phụ**      | Rất ít / không đáng kể.                                                                                         | Cần thêm dung lượng trống **$\approx$ kích thước bảng hiện tại** trong quá trình rewrite. | Không.                                                        |
| **Tuân thủ MVCC**        | Có.                                                                                                             | Có.                                                                                       | **Không** (vi phạm ngữ nghĩa MVCC nghiêm ngặt).               |
| **Khuyến nghị vận hành** | Chạy định kỳ/tự động để giữ kích thước bảng ổn định; **ưu tiên số 1**.                                          | Chỉ dùng khi bảng phình to (_bloat_) nghiêm trọng sau xóa hàng loạt.                      | Dùng khi cần dọn sạch hoàn toàn bảng tạm/staging.             |
## Thu Thập Thống Kê (Updating Planner Statistics - ANALYZE)

- **Phương thức hoạt động**: Dùng **lấy mẫu ngẫu nhiên** (_statistical random sampling_), không quét tuần tự toàn bảng $\rightarrow$ chạy rất nhanh.
    
- **Quy tắc điều chỉnh**:
    
    - Các cột thường dùng trong `WHERE`/`JOIN` có phân bố không đều $\rightarrow$ tăng chi tiết histogram qua `ALTER TABLE ... SET STATISTICS <n>` (hoặc biến `default_statistics_target`).
        
    - Hàm tính toán trong truy vấn $\rightarrow$ tạo **Extended Statistics** hoặc **Expression Index** để optimizer thu thập thống kê hàm.
        
- **Điểm mù của Autovacuum đối với ANALYZE**:
    
    - **Bảng phân vùng (Partitioned tables)**: Autovacuum chỉ phân tích các partition con, **không tự chạy ANALYZE trên bảng cha** $\rightarrow$ phải lên lịch chạy thủ công `ANALYZE` trên bảng cha.
        
    - **Foreign tables**: Autovacuum bỏ qua hoàn toàn.
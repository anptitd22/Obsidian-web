---
title: WAL
tags:
  - PostgreSQL
  - RDBMS
---
## Core Principle

- **Nguyên tắc "Ghi Log Trước - Ghi Data Sau":** Mọi thay đổi trên bảng và chỉ mục (Data Pages / Heap) chỉ được phép ghi xuống đĩa **sau khi** bản ghi nhật ký (WAL records) mô tả các thao tác đó đã được xả an toàn vào bộ nhớ vĩnh viễn (persistent storage).
    
- **Mục tiêu chính:** Đảm bảo toàn vẹn dữ liệu, thỏa mãn tính bền vững (**Durability**) và nguyên tử (**Atomicity**) của giao dịch theo chuẩn ACID.
    
- **Cơ chế phục hồi REDO (Roll-Forward Recovery):** Khi gặp sự cố sập nguồn/crash, hệ thống không bị mất dữ liệu vì mọi thay đổi chưa kịp ghi vào trang dữ liệu đều có thể được đọc và thực hiện lại (_replay_) từ WAL.

```
[Thay đổi dữ liệu trong RAM]
           │
           ├── 1. Ghi & Flush bắt buộc ──► [WAL Files trên Đĩa] (Tuần tự / Nhanh)
           │
      (Commit thành công)
           │
           └── 2. Flush trì hoãn (Lazy) ─► [Data Files trên Đĩa] (Ngẫu nhiên / Chậm)
```

## Lợi Ích Vận Hành & Tối Ưu Hiệu Năng (Performance Gains)

- **Chuyển Random I/O thành Sequential I/O:**
    
    - Không cần flush các trang dữ liệu phân tán (Random Write) mỗi khi `COMMIT`.
        
    - Chỉ cần ghi tệp WAL theo dạng nối tiếp tuần tự (**Append-only / Sequential Write**), giúp giảm đáng kể chi phí trễ đĩa (_disk head movement / write amplification_).
        
- **Nhóm Cam Kết (Group Commit):**
    
    - Khi có nhiều giao dịch nhỏ chạy đồng thời, hệ thống chỉ cần **một thao tác `fsync` tệp WAL duy nhất** để xác nhận thành công cho nhiều giao dịch cùng lúc.
        
- **Hạn chế đệm kép với Journaling File System:**
    
    - Bản thân WAL đã đóng vai trò cơ chế khôi phục tệp cơ sở dữ liệu sau crash, do đó không cần chế độ ghi nhật ký toàn bộ dữ liệu của hệ điều hành.
        
    - _Tối ưu hóa:_ Có thể gắn ổ đĩa chứa DB với chế độ `data=writeback` (ví dụ trên Linux ext3/ext4) để tắt việc hệ điều hành ghi nhật ký dữ liệu hai lần, giảm áp lực I/O.

## Ứng Dụng Trong Sao Lưu & Khôi Phục (Backup & PITR)

- **Khôi phục về một thời điểm cụ thể (Point-in-Time Recovery - PITR):**
    
    - Cài đặt một bản sao lưu vật lý cơ sở (_Base Backup_).
        
    - Phát lại các tệp WAL tích lũy từ thời điểm backup đến đúng thời điểm mong muốn (ví dụ: ngay trước khi có lệnh `DROP TABLE` nhầm lẫn).
        
- **Online/Fuzzy Backup (Sao lưu trực tuyến không khóa):**
    
    - Bản sao lưu vật lý không bắt buộc phải là ảnh chụp đóng băng tức thời (_instantaneous consistent snapshot_).
        
    - Bản backup có thể kéo dài trong một khoảng thời gian; việc phát lại toàn bộ WAL sinh ra trong suốt thời gian backup sẽ tự động đồng bộ và sửa chữa mọi trạng thái bất nhất nội bộ.

>[!note]
>WAL (postgresql) = binlog (Mysql) + Redo log (Mysql)


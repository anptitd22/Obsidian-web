---
title: Isolation levels
tags:
  - MySQL
  - Topic
  - InnoDB
---
| **Isolation Level**                           | **Dirty Read** | **Non-repeatable Read** | **Phantom Read** | **Cơ chế xử lý chính**                                                                                                                   |
| --------------------------------------------- | -------------- | ----------------------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **READ UNCOMMITTED**                          | Có             | Có                      | Có               | Đọc trực tiếp dữ liệu mới nhất trên bộ đệm (kể cả chưa commit). Không tạo bản snapshot MVCC.                                             |
| **READ COMMITTED**                            | Không          | Có                      | Có               | Dùng MVCC: Mỗi câu lệnh `SELECT` tạo ra một **Snapshot (Read View) mới**.                                                                |
| **REPEATABLE READ**<br>_(Mặc định của MySQL)_ | Không          | Không                   | Có               | Dùng MVCC (tạo Read View ở câu lệnh `SELECT` đầu tiên) kết hợp **Next-Key Locks** (Record lock + Gap lock) cho Locking Reads.            |
| **SERIALIZABLE**                              | Không          | Không                   | Không            | Tự động chuyển mọi câu lệnh `SELECT` thuần túy thành `SELECT ... FOR SHARE` nếu `autocommit = 0`. Khóa hoàn toàn các giao dịch xung đột. |

Liên quan:
- [[DE/Databases/Topic/Isolation levels|Isolation levels]]
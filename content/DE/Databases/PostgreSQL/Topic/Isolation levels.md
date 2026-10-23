---
title: Isolation levels
tags:
  - PostgreSQL
  - Topic
---
| **Isolation Level**                                           | **Dirty Read** | **Dirty Read**                           | **Non-repeatable Read** | **Phantom Read** | **Serialization Anomaly** | **Cơ chế xử lý chính**                                                                                                                                                                                                                      |
| ------------------------------------------------------------- | -------------- | ---------------------------------------- | ----------------------- | ---------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **READ UNCOMMITTED**                                          | Có             | **Không** _(tự nâng lên Read Committed)_ | Có                      | Có               | Có                        | Postgres không hỗ trợ Dirty Read; cấu hình mức này thực chất chạy như **Read Committed**.                                                                                                                                                   |
| **READ COMMITTED**<br><br>  <br><br>_(Mặc định của Postgres)_ | Không          | **Không**                                | Có                      | Có               | Có                        | Dùng MVCC: Mỗi câu lệnh SQL (`SELECT`) tạo ra một **Snapshot mới** tại thời điểm câu lệnh đó bắt đầu chạy.                                                                                                                                  |
| **REPEATABLE READ**                                           | Không          | **Không**                                | **Không**               | **Không**        | Có                        | Dùng MVCC: Tạo một **Snapshot duy nhất** tại thời điểm câu lệnh đầu tiên trong transaction chạy; snapshot này giữ nguyên đến khi transaction kết thúc. Báo lỗi `could not serialize access` nếu xảy ra xung đột ghi (First-committer-wins). |
| **SERIALIZABLE**                                              | Không          | **Không**                                | **Không**               | **Không**        | **Không**                 | Sử dụng thuật toán **SSI (Serializable Snapshot Isolation)** thông qua các khóa dạng vị từ (**SIREAD lock**) để phát hiện chu kỳ phụ thuộc (rw-antidependencies) và abort giao dịch vi phạm.                                                |

Liên quan:
- [[DE/Databases/Topic/Isolation levels|Isolation levels]]

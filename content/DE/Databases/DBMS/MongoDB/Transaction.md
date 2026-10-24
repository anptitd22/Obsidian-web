---
title: Transaction
tags:
  - MongoDB
  - RDBMS
---
## Bản Chất & Thiết Kế Dữ Liệu (Core Concepts & Philosophy)

- **Single-document Atomicity:** Thao tác trên 1 document đơn lẻ **luôn có tính nguyên tử (atomic)** mặc định.
    
- **Quy tắc thiết kế Schema:**
    
    - Ưu tiên mô hình phi chuẩn hóa (**Denormalization - embedded docs & arrays**) để giải quyết bài toán quan hệ mà không cần multi-document transactions.
        
    - Multi-document transactions mang chi phí hiệu năng (**performance overhead**) lớn hơn nhiều so với single-doc writes $\rightarrow$ **chỉ dùng khi thực sự bắt buộc**, không dùng thay thế cho việc thiết kế schema hợp lý.
        
- **Phạm vi hỗ trợ:** Hoạt động trên nhiều collections, databases, documents, và trải rộng trên cả **Replica Sets (từ MongoDB 4.0)** và **Sharded Clusters (từ MongoDB 4.2)**.


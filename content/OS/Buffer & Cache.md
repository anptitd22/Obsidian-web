**Buffer (Bộ đệm chuyển tiếp):** Đóng vai trò là "trạm trung chuyển" giải quyết sự lệch pha về tốc độ hoặc kích thước dữ liệu giữa hai thiết bị/tiến trình (ví dụ: giữa ổ đĩa và RAM, hoặc giữa mạng và màn hình).

**Cache (Bộ nhớ đệm tăng tốc):** Đóng vai trò là "kho lưu trữ tắt" giúp giảm thời gian truy cập bằng cách giữ lại các dữ liệu thường xuyên được dùng lại.

### So sánh chi tiết Buffer và Cache

| **Tiêu chí**              | **Buffer**                                                                   | **Cache**                                                               |
| ------------------------- | ---------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| **Mục đích chính**        | Đồng bộ hóa tốc độ giữa hai hệ thống chênh lệch hiệu năng.                   | Tăng tốc độ truy xuất bằng cách tái sử dụng dữ liệu.                    |
| **Tần suất dùng dữ liệu** | Dữ liệu thường chỉ được đọc/ghi **một lần** rồi xóa (hàng đợi).              | Dữ liệu được lưu trữ để **đọc lại nhiều lần**.                          |
| **Bản chất hoạt động**    | Lưu dữ liệu đang trong quá trình di chuyển (In-transit).                     | Lưu bản sao của dữ liệu gốc để truy cập nhanh hơn.                      |
| **Ví dụ thực tế**         | Xem video YouTube (đoạn video tải trước để không bị giật lag khi mạng chậm). | Cache trình duyệt (lưu ảnh, CSS của trang web để lần sau mở nhanh hơn). |
| **Trường hợp áp dụng**    | In ấn (Print Spooler), ghi file ra đĩa, stream âm thanh/video.               | CPU Cache (L1/L2/L3), CDN, Redis, database query caching.               |

**Cách ghi nhớ nhanh:**

- **Buffer** giúp quá trình truyền tải **liên tục, không bị đứt đoạn** (tập trung vào sự mượt mà).

- **Cache** giúp quá trình lấy dữ liệu **nhanh hơn** cho các lần sau (tập trung vào tốc độ).
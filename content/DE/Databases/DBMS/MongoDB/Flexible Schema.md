---
title: Flexible Schema
tags:
  - MongoDB
  - RDBMS
---
## Sự linh hoạt về cấu trúc (Polymorphism & Dynamic Schema)

- **Không cần khai báo trước:** Bạn không cần định nghĩa bảng, cột hay kiểu dữ liệu cố định như trong các cơ sở dữ liệu quan hệ (SQL).
    
- **Đa hình (Polymorphism):** Các tài liệu nằm trong cùng một tập hợp (collection) có thể chứa các trường (field) khác nhau.
    
- **Thêm trường dễ dàng:** Muốn bổ sung thông tin mới? Bạn chỉ cần ghi dữ liệu mới vào mà không cần chỉnh sửa hệ thống, không làm gián đoạn ứng dụng (**zero downtime**) và không cần cập nhật các bộ ánh xạ ORM.
    
- **Tích hợp nguồn dữ liệu đa dạng:** Đặc tính này rất hữu ích khi bạn hợp nhất dữ liệu từ nhiều nguồn khác nhau hoặc liên tục nâng cấp tính năng ứng dụng theo thời gian.
    
## Kiểm soát cấu trúc khi cần (Schema Validation)

- Tuy linh hoạt là thế mạnh, nhưng tính linh hoạt quá mức đôi khi gây rủi ro về chất lượng dữ liệu.
    
- Để giải quyết vấn đề này, MongoDB cung cấp tính năng **Schema Validation**. Bạn có thể thiết lập các quy tắc kiểm tra (ví dụ: bắt buộc phải có trường `email`, tuổi phải là `int`) cho từng tập hợp để đảm bảo dữ liệu luôn đạt chuẩn quản trị.

### Ví dụ thực tế: Hệ thống quản lý sản phẩm cho cửa hàng E-Commerce

Giả sử bạn có một tập hợp tên là `products` (sản phẩm). Cửa hàng của bạn bán 3 loại mặt hàng: **Áo thun**, **Điện thoại**, và **Sách**.

1. Tạo Schema Validation cho cả 3 loại sản phẩm

Dưới đây là lệnh tạo collection `products` có quy tắc kiểm tra chi tiết:

```sql
db.createCollection("products", {
   validator: {
      $jsonSchema: {
         bsonType: "object",
         required: ["name", "price", "category"], // Cả 3 loại BẮT BUỘC phải có 3 trường này
         properties: {
            name: { bsonType: "string" },
            price: { bsonType: ["int", "long", "decimal128"] },
            category: { enum: ["clothing", "electronics", "books"] }
         },
         
         // ĐOẠN NÀY DÙNG ĐỂ BIẾT KHI NÀO LÀ ÁO THUN, ĐIỆN THOẠI HAY SÁCH
         allOf: [
            // Quy tắc 1: Nếu là ÁO THUN (clothing) -> Bắt buộc có sizes, colors
            {
               if: { properties: { category: { const: "clothing" } } },
               then: { required: ["sizes", "colors"] }
            },
            // Quy tắc 2: Nếu là ĐIỆN THOẠI (electronics) -> Bắt buộc có ram, storage
            {
               if: { properties: { category: { const: "electronics" } } },
               then: { required: ["ram", "storage"] }
            },
            // Quy tắc 3: Nếu là SÁCH (books) -> Bắt buộc có author, pages
            {
               if: { properties: { category: { const: "books" } } },
               then: { required: ["author", "pages"] }
            }
         ]
      }
   }
})
```

2. Các Use Case Insert (Thêm dữ liệu thực tế)

Sau khi tạo Schema Validation ở trên, hãy xem MongoDB xử lý các câu lệnh `insert` như thế nào:

#### Use Case 1: Insert ÁO THUN (Thành công)

MongoDB nhìn vào `category: "clothing"` -> Kích hoạt quy tắc 1 -> Kiểm tra thấy có đủ `sizes` và `colors`.

```sql
db.products.insertOne({
  "name": "Áo thun Polo",
  "price": 200000,
  "category": "clothing",
  "sizes": ["M", "L"],
  "colors": ["Đen"]
})
// RESULT: Ghi thành công!
```

#### Use Case 2: Insert ĐIỆN THOẠI nhưng quên nhập Dung lượng bộ nhớ (Thất bại)

MongoDB nhìn vào `category: "electronics"` -> Kích hoạt quy tắc 2 -> Phát hiện **thiếu trường `storage`**.

```sql
db.products.insertOne({
  "name": "iPhone 15 Pro",
  "price": 25000000,
  "category": "electronics",
  "ram": "8GB"
  // Thiếu "storage"!
})
// RESULT: Báo lỗi "Document failed validation" và bị TỪ CHỐI!
```

#### Use Case 3: Insert SÁCH nhưng gõ nhầm Số trang thành dạng Chữ (Thất bại)

MongoDB nhìn vào `category: "books"` -> Kích hoạt quy tắc 3 -> Kiểm tra thấy trường `pages` bị sai kiểu dữ liệu (truyền chuỗi `"320 trang"` thay vì số `320`).

```sql
db.products.insertOne({
  "name": "Đắc Nhân Tâm",
  "price": 90000,
  "category": "books",
  "author": "Dale Carnegie",
  "pages": "320 trang" // Bị sai! Phải là số integer (320)
})
// RESULT: Báo lỗi "Document failed validation" và bị TỪ CHỐI!
```
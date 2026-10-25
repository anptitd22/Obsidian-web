| **Loại Index** | **Cấu trúc dữ liệu**                      | **Khả năng truy vấn chính**                   | **Kích thước index** | **Chi phí ghi (Write Overhead)** |
| -------------- | ----------------------------------------- | --------------------------------------------- | -------------------- | -------------------------------- |
| **B-Tree**     | Cây cân bằng B-Tree                       | `=`, phạm vi (`<`, `>`), sắp xếp (`ORDER BY`) | Trung bình           | Thấp                             |
| **Hash**       | Bảng băm (Hash bucket)                    | Duy nhất so bằng `=`                          | Nhỏ - Trung bình     | Thấp                             |
| **GiST**       | Cây tổng quát / R-Tree                    | Tọa độ, hình học, phạm vi dải giá trị, k-NN   | Lớn                  | Trung bình - Cao                 |
| **SP-GiST**    | Cây phân vùng không gian (Trie, Quadtree) | Tiền tố chuỗi, tọa độ dữ liệu phân bố lệch    | Trung bình           | Trung bình                       |
| **GIN**        | Chỉ mục đảo ngược                         | Tìm kiếm từ khóa, mảng, thuộc tính JSONB      | Lớn                  | Cao                              |
| **BRIN**       | Tóm tắt Min/Max theo block                | Quét phạm vi trên dữ liệu tuần tự vật lý      | Cực nhỏ              | Rất thấp                         |

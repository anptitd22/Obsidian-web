## Normalization

**Định nghĩa:** Quá trình tổ chức lại các bảng và thuộc tính nhằm **giảm thiểu/loại bỏ dữ liệu dư thừa** (_data redundancy_) và đảm bảo **tính toàn vẹn dữ liệu** (_data integrity_).

**Mục tiêu chính:**

- Giảm kích thước lưu trữ không cần thiết.
    
- Đơn giản hóa việc quản lý và mở rộng hệ thống.
    
- Triệt tiêu **3 loại bất thường (Anomalies)** khi thao tác dữ liệu:
    
    - **Insertion Anomaly (Lỗi chèn):** Không thể thêm một đối tượng độc lập nếu thiếu dữ liệu của đối tượng khác (VD: Không thêm được phòng ban mới nếu chưa có nhân viên nào trực thuộc).
        
    - **Update Anomaly (Lỗi sửa):** Một thông tin bị lặp ở nhiều dòng; cập nhật thiếu sót khiến dữ liệu mâu thuẫn (VD: Đổi địa điểm phòng ban phải sửa ở tất cả nhân viên thuộc phòng đó).
        
    - **Deletion Anomaly (Lỗi xóa):** Xóa một thực thể vô tình làm mất luôn thông tin của thực thể khác (VD: Xóa hết nhân viên IT làm mất luôn thông tin phòng ban IT).

### Normal Forms in DBMS

| **Dạng chuẩn** | **Điều kiện kế thừa** | **Điều kiện tiên quyết (Khái niệm cốt lõi)**                                                                                  | **Mục tiêu triệt tiêu**                   |
| -------------- | --------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| **1NF**        | Không                 | Giá trị tại mỗi ô là **nguyên tử** (Atomic / Single-valued), không chứa mảng hay danh sách                                    | Dữ liệu đa trị, thuộc tính lặp lại        |
| **2NF**        | $\ge \text{1NF}$      | Mọi thuộc tính không khóa phải **phụ thuộc hàm đầy đủ** vào toàn bộ khóa chính                                                | Phụ thuộc bộ phận (Partial Dependency)    |
| **3NF**        | $\ge \text{2NF}$      | Không có **phụ thuộc bắc cầu**; hoặc với mọi $X \rightarrow Y$ không tầm thường: $X$ là siêu khóa HOẶC $Y$ là thuộc tính khóa | Phụ thuộc bắc cầu (Transitive Dependency) |
| **BCNF**       | $\ge \text{3NF}$      | Chặt hơn 3NF: Với mọi $X \rightarrow Y$ không tầm thường, **$X$ bắt buộc phải là siêu khóa**                                  | Bất thường do khóa ứng viên chồng chéo    |
| **4NF**        | $\ge \text{BCNF}$     | Không tồn tại **phụ thuộc đa trị** ($X \twoheadrightarrow Y$) độc lập                                                         | Phụ thuộc đa trị (Multivalued Dependency) |
| **5NF**        | $\ge \text{4NF}$      | Không thể phân rã không mất mát thông tin thêm được nữa (**Join Dependency**)                                                 | Phụ thuộc kết nối (Join Dependency)       |
#### Dạng Chuẩn 1 (1NF - First Normal Form)

- **Quy tắc:** Mỗi ô dữ liệu chỉ chứa **duy nhất một giá trị** (nguyên tử - _atomic value_). Cột không được chứa list, set hoặc nested record.
    
- **Khắc phục:** Tách các giá trị phân tách bằng dấu phẩy thành nhiều hàng hoặc chuyển sang bảng phụ quan hệ $1-N$.

#### Dạng Chuẩn 2 (2NF - Second Normal Form)

- **Quy tắc:** Đạt 1NF VÀ **không tồn tại phụ thuộc hàm bộ phận (No Partial Functional Dependency)**.
    
    - Mọi thuộc tính không khóa (_non-prime attribute_) phải phụ thuộc vào **toàn bộ** khóa chính (Composite Primary Key), không được phụ thuộc vào một phần của khóa.
        
    - _Lưu ý:_ Nếu khóa chính của bảng chỉ gồm **1 cột đơn lẻ**, bảng đó tự động thỏa mãn 2NF (nếu đã ở 1NF).
        
- **Khắc phục:** Tách các thuộc tính phụ thuộc vào một phần của khóa ra bảng riêng.

#### Dạng Chuẩn 3 (3NF - Third Normal Form)

- **Quy tắc:** Đạt 2NF VÀ **không tồn tại phụ thuộc bắc cầu (No Transitive Dependency)**.
    
    - Thuộc tính không khóa không được phụ thuộc vào thuộc tính không khóa khác ($A \rightarrow B \rightarrow C$).
        
    - **Điều kiện toán học:** Với mọi phụ thuộc hàm không tầm thường $X \rightarrow Y$:
        
        1. $X$ là một **siêu khóa (Super Key)**, HOẶC
            
        2. $Y$ là một **thuộc tính khóa (Prime Attribute)**.
            
- **Khắc phục:** Tách các thuộc tính bắc cầu ($B \rightarrow C$) thành một bảng riêng (VD: tách thông tin chi tiết phòng ban ra khỏi bảng nhân viên).

#### Dạng Chuẩn Boyce-Codd (BCNF - 3.5NF)

**Quy tắc:** Dạng nâng cao, chặt chẽ hơn của 3NF.

- Với **mọi** phụ thuộc hàm không tầm thường $X \rightarrow Y$, **$X$ bắt buộc phải là một Siêu Khóa (Super Key)**.
    
- BCNF loại bỏ trường hợp ngoại lệ thứ 2 của 3NF ($Y$ là thuộc tính khóa). BCNF cần thiết khi bảng có nhiều khóa ứng viên (_candidate keys_) bị chồng lấn/giao nhau về mặt thuộc tính.

#### Dạng Chuẩn 4 (4NF) & Dạng Chuẩn 5 (5NF)

- **4NF (Fourth Normal Form):** Đạt BCNF + Không chứa **phụ thuộc đa trị (Multivalued Dependency - MVD)** độc lập.
    
    - Áp dụng khi 1 thực thể có 2 mối quan hệ $1-N$ hoàn toàn độc lập nằm chung 1 bảng (VD: 1 nhân viên vừa có nhiều kỹ năng, vừa có nhiều ngôn ngữ $\rightarrow$ bảng bị nhân tích Descartes).
        
- **5NF (Fifth Normal Form / Project-Join Normal Form - PJNF):** Đạt 4NF + Thỏa mãn **Join Dependency**.
    
    - Bảng không thể phân rã thành các bảng nhỏ hơn mà khi Join lại bị dư thừa bản ghi giả (_spurious tuples_). Hầu hết các hệ thống OLTP thực tế chỉ cần chuẩn hóa đến 3NF hoặc BCNF là tối ưu.

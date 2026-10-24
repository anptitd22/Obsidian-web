## Logical Join & Physical Join

- **Logical Join (Mức khai báo logic):**
    
    - Định nghĩa bởi câu lệnh SQL: `INNER JOIN`, `LEFT JOIN`, `RIGHT JOIN`, `FULL OUTER JOIN`, `CROSS JOIN`.
        
    - Chỉ ra kết quả logic cần giữ lại theo ngữ nghĩa quan hệ.
        
- **Physical Join (Mức thực thi vật lý):**
    
    - Thuật toán cụ thể do Query Optimizer lựa chọn bên dưới engine: **Nested Loop Join**, **Hash Join**, **Merge Join**.
        
    - Optimizer quyết định thuật toán dựa trên:
        
        - Kích thước các bảng (Cardinarity / Tuples count).
            
        - Sự tồn tại của Index trên cột join.
            
        - Trạng thái dữ liệu đã sắp xếp (Sorted) hay chưa.
            
        - Tài nguyên bộ nhớ RAM cho phép (`work_mem`, `hash_mem_multiplier`).
            
        - Toán tử so sánh: Toán tử bằng (`=`) hay toán tử khoảng (`<`, `<=`, `>`, `>=`).

---
## Bảng So Sánh Chi Tiết 3 Thuật Toán Join

| **Tiêu chí**                | **Nested Loop Join**                                                              | **Hash Join**                                                                                     | **Merge Join (Sort-Merge)**                                                             |
| --------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **Bản chất hoạt động**      | Duyệt từng hàng bảng ngoài (Outer), tìm kiếm khớp trên bảng trong (Inner).        | **Giai đoạn Build:** Băm bảng nhỏ vào RAM.<br><br>**Giai đoạn Probe:** Quét bảng lớn dò bảng băm. | Sắp xếp 2 bảng theo join key rồi đồng thời quét song song 2 con trỏ.                    |
| **Yêu cầu dữ liệu sắp xếp** | Không                                                                             | Không                                                                                             | **Bắt buộc phải sắp xếp** (trực tiếp hoặc qua Index)                                    |
| **Yêu cầu Index**           | Rất cần Index trên Join Key của bảng Inner (**Index Nested Loop**).               | Không cần Index.                                                                                  | Không bắt buộc, nhưng có Index B+Tree sẵn sẽ bỏ qua được bước Sort.                     |
| **Tiêu tốn RAM**            | Rất ít (gần như $O(1)$ bộ nhớ đệm).                                               | **Nhiều nhất** (cần đủ RAM chứa Hash Table của bảng nhỏ).                                         | Trung bình (cần RAM cho bước Sort nếu dữ liệu chưa sắp xếp).                            |
| **Toán tử hỗ trợ**          | Mọi toán tử (`=`, `<`, `>`, `<=`, `>=`).                                          | **Chỉ tối ưu cho phép toán bằng (`=`)**. Không dùng cho range join.                               | Tối ưu cho phép bằng (`=`) và rất tốt cho **phép toán phạm vi (`<`, `>`)**.             |
| **Kịch bản tối ưu nhất**    | Một bảng rất nhỏ (hoặc được filter còn vài chục dòng) join với bảng lớn có index. | Hai bảng lớn, **không có index**, dữ liệu chưa sắp xếp, điều kiện so sánh bằng (`=`).             | Hai bảng lớn, cả hai **đều đã được sort sẵn** theo join key (hoặc có sẵn B-Tree Index). |
| **Độ phức tạp thời gian**   | $O(M \times \log N)$ (có index)<br><br>$O(M \times N)$ (quét cạn - Naive)         | $O(M + N)$ (trường hợp lý tưởng bảng băm vừa trong RAM)                                           | $O(M + N)$ (nếu đã sort)<br><br>$O(M \log M + N \log N)$ (nếu phải sort)                |

### Nested Loop Join (Vòng lặp lồng nhau)

- **Ý tưởng:** Giống 2 vòng lặp lồng nhau (`for row_r in R: for row_s in S:`).
    
- **Các biến thể:**
    
    1. _Naive Nested Loop Join:_ Quét cạn bảng trong cho mỗi dòng bảng ngoài $\rightarrow$ Cực chậm trên bảng lớn ($O(M \times N)$).
        
    2. _Index Nested Loop Join:_ Với mỗi dòng bảng ngoài, dùng B-Tree Index trên bảng trong để tìm nhanh $\rightarrow$ Tốc độ $O(M \times \log N)$.
        
    3. _Block Nested Loop Join:_ Đọc từng block trang dữ liệu của bảng ngoài vào bộ nhớ để giảm số lần đọc I/O đĩa của bảng trong.
        
- **Lưu ý thực tế:** Không phải lúc nào cũng dành cho "1 bảng to + 1 bảng nhỏ". Nó chỉ thực sự nhanh khi **kết quả sau lọc (filtered rows)** của bảng ngoài nhỏ và bảng trong có index hiệu quả.

### Hash Join (Nối băm)

- **Quy trình 2 giai đoạn:**
    
    1. **Build Phase:** Quét toàn bộ bảng nhỏ hơn (Driving table), áp dụng hàm băm $h(\text{key})$ để đưa các hàng vào các bucket trong bộ nhớ RAM (Hash Table).
        
    2. **Probe Phase:** Quét tuần tự bảng lớn hơn (Probing table), tính $h(\text{key})$ cho từng dòng và tra cứu trực tiếp vào bucket tương ứng trong Hash Table để nối dữ liệu.
        
- **Các biến thể xử lý khi thiếu RAM:**
    
    - _Classic Hash Join:_ Hash table vừa trọn vẹn trong RAM (`work_mem`).
        
    - _Grace Hash Join / Hybrid Hash Join:_ Khi bảng nhỏ vượt quá RAM, dữ liệu bị băm và phân vùng (partition) tạm xuống đĩa, sau đó nạp từng cặp partition vào RAM để join.
        
- **Hạn chế:** Chỉ hỗ trợ **Equi-join** (`ON A.id = B.id`). Không áp dụng được cho các điều kiện so sánh khoảng (`ON A.val >= B.val`).

### Merge Join / Sort-Merge Join (Nối hợp nhất)

- **Quy trình 2 giai đoạn:**
    
    1. **Sort Phase:** Đảm bảo cả hai quan hệ đều được sắp xếp tăng dần/giảm dần theo join key.
        
    2. **Merge Phase:** Đặt 2 con trỏ ở đầu 2 bảng. So sánh giá trị:
        
        - Nếu giá trị bằng nhau: Tạo kết quả join, dịch con trỏ.
            
        - Nếu giá trị con trỏ A nhỏ hơn con trỏ B: Tịnh tiến con trỏ A.
            
        - Nếu giá trị con trỏ B nhỏ hơn con trỏ A: Tịnh tiến con trỏ B.
            
- **Điểm mạnh cốt lõi:**
    
    - Xử lý dữ liệu lớn xuất sắc mà **không ngốn RAM đột biến** như Hash Join (vì đọc tuần tự dạng stream).
        
    - Xử lý tốt các điều kiện bất đẳng thức hoặc phạm vi.
        
    - Nếu cột join đã có sẵn clustered index hoặc B-Tree index, optimizer bỏ qua hoàn toàn giai đoạn Sort.

## Cheat Sheet Ra Quyết Định Của Query Optimizer

```
Điều kiện Join có phải là dấu bằng (=) không?
│
├── KHÔNG (>, <, >=, <=)
│     ├── Bảng trong có Index? ──► YES: Nested Loop Join
│     └── Bảng trong không Index? ─► YES: Merge Join
│
└── CÓ (=)
      ├── Cả 2 bảng đã được Sort (hoặc có B-Tree Index)? ──► YES: Merge Join
      ├── Bảng ngoài nhỏ + Bảng trong có Index? ──────────► YES: Index Nested Loop Join
      └── 2 bảng lớn, chưa sort, không có Index phù hợp? ──► YES: Hash Join (Build bảng nhỏ, Probe bảng lớn)
```

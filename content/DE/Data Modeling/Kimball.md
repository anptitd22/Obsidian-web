## 1. Nền Tảng DW/BI & Dimensional Modeling (Chương 1 & 3)

### Ẩn dụ cốt lõi

- **Nhà hàng:** Hệ thống ETL là **nhà bếp** (chuẩn bị, sơ chế, làm sạch dữ liệu thô). Presentation Layer & BI là **phòng ăn** (phục vụ dữ liệu sạch, trực quan, dễ tiêu thụ).
    
- **Xuất bản:** Quản lý DW/BI giống **tổng biên tập** — tập trung vào nhu cầu độc giả (business user) thay vì tự thỏa mãn bằng độ phức tạp của công nghệ.
    

### Hai thành phần cốt lõi

- **Fact Table:** Chứa các phép đo định lượng (thường là số), sinh ra từ một sự kiện nghiệp vụ.
    
- **Dimension Table:** Chứa ngữ cảnh mô tả (_Who, What, Where, When, Why, How_). Được phi chuẩn hóa (denormalized) để phục vụ `GROUP BY` và `WHERE` mà không cần join nhiều bảng.
    

### So sánh: Star Schema vs. OLAP Cube

| **Tiêu chí**              | **Star Schema (RDBMS)**                                               | **OLAP Cube (Multidimensional)**                       |
| ------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------ |
| **Lưu trữ & Ngôn ngữ**    | RDBMS / Columnar; truy vấn bằng **SQL**                               | Database đa chiều; truy vấn bằng **MDX / XMLA**        |
| **Tính toán**             | Tính toán tại thời điểm query (join on-the-fly)                       | Pre-aggregated sẵn; tính toán cực nhanh                |
| **Xử lý SCD / Overwrite** | Rất mượt mà với SCD 1/3 và Accumulating Snapshot                      | Kém; ghi đè/cập nhật ép Cube phải reprocess            |
| **Phân cấp phức tạp**     | Khó xử lý Ragged Hierarchy (cần bridge table)                         | Hỗ trợ cú pháp native cho Ragged Hierarchy             |
| **Role-playing**          | Rất dễ (dùng SQL view/alias)                                          | Kém linh hoạt; thường phải nhân bản physical dimension |
| **Mối quan hệ**           | **Nền tảng ổn định nhất** (atomic grain) để cấp dữ liệu lên OLAP Cube |                                                        |

## 2. Quy Trình Thiết Kế Mô Hình Chiều 4 Bước (Chương 2, 3, 18)

Quy trình bất biến cho mọi dự án Kimball:

```
[B1: Chọn Quy trình Nghiệp vụ] 
       ↓
[B2: Tuyên bố Mức độ Hạt (Declare the Grain)]  ← BƯỚC QUAN TRỌNG NHẤT
       ↓
[B3: Xác định các Chiều (Dimensions)]
       ↓
[B4: Xác định các Thước đo (Facts)]
```

- **Bước 1 — Business Process:** Xác định hoạt động tác nghiệp sinh ra dữ liệu (ví dụ: quét hóa đơn, giao hàng), diễn đạt bằng động từ. **Thiết kế theo quy trình, không theo phòng ban**.
    
- **Bước 2 — Declare the Grain:** Trả lời chính xác: _"Một dòng trong bảng Fact đại diện cho cái gì?"_.
    
    - _Quy tắc:_ Luôn ưu tiên **atomic grain** (mức chi tiết nhất có thể). **Tuyệt đối không trộn lẫn các mức hạt khác nhau trong cùng một Fact table**.
        
- **Bước 3 — Dimensions:** Xác định toàn bộ ngữ cảnh bao quanh hạt đã chọn ở Bước 2.
    
- **Bước 4 — Facts:** Xác định các chỉ số đo lường. Mọi Fact phải tương thích 100% với mức hạt ở Bước 2.
    

## 3. Phân Loại Fact & Dimension Kỹ Thuật

### 3.1. Phân loại Fact Tables

- **Transaction Fact:** Mỗi dòng là một sự kiện tại một thời điểm (vd: 1 dòng item trên hóa đơn POS). Dữ liệu chi tiết nhất.
    
- **Periodic Snapshot Fact:** Đo lường trạng thái tại các mốc thời gian đều đặn (vd: số dư tài khoản cuối ngày, tồn kho mỗi tuần).
    
- **Accumulating Snapshot Fact:** Một dòng đại diện cho toàn bộ vòng đời của một quy trình (vd: Đặt hàng → Đóng gói → Giao hàng → Thu tiền). Chứa nhiều cột mốc ngày (Role-playing Date) và các cột đo độ trễ (Lags/Durations); được cập nhật liên tục.
    
- **Factless Fact Table:** Không chứa số đo, chỉ chứa các foreign keys. Dùng để:
    
    - Theo dõi sự kiện (Event tracking: sinh viên điểm danh).
        
    - Đo độ phủ/điều kiện (Coverage: sản phẩm nào có khuyến mãi ở cửa hàng nào vào ngày nào).
        

### 3.2. Tính cộng gộp của Fact

- **Additive (Cộng gộp toàn phần):** Cộng được qua mọi chiều (vd: Doanh số, Số lượng bán).
    
- **Semi-additive (Bán cộng gộp):** Không cộng được qua chiều Thời gian (vd: Số dư ngân hàng, Tồn kho — chỉ lấy trung bình hoặc giá trị cuối kỳ).
    
- **Non-additive (Không cộng gộp):** Các tỉ lệ, đơn giá (vd: Biên lợi nhuận %). Phải lưu tử số và mẫu số riêng, tính lại ở tầng BI/SQL.
    

### 3.3. Các loại Dimension đặc thù

- **Surrogate Key (Khóa thay thế):** Số nguyên tự tăng vô nghĩa (`1, 2, 3...`), **bắt buộc** dùng làm Primary Key cho Dimension. Tránh dùng Natural Key từ source để hỗ trợ SCD và xử lý khóa tái sử dụng.
    
- **Degenerate Dimension (Chiều suy biến):** Mã định danh giao dịch (Số hóa đơn, Mã vận đơn) nằm trực tiếp trong Fact table mà không có bảng Dimension riêng.
    
- **Junk Dimension (Chiều rác):** Bảng gom các cờ trạng thái, cờ boolean (Flags/Indicators) rời rạc lại thành một bảng duy nhất để tránh tạo quá nhiều Dimension nhỏ.
    
- **Outrigger Dimension:** Một Dimension tham chiếu tới một Dimension thứ cấp (vd: Dim_Customer trỏ đến Dim_County). **Hạn chế dùng** vì tạo hiệu ứng Snowflake làm chậm query.
    
- **Role-Playing Dimension:** Một bảng vật lý đóng nhiều vai trò khác nhau trong cùng Fact table (vd: Dim_Date dùng cho `Order_Date`, `Ship_Date`, `Payment_Date` thông qua Database Views / Aliases).
    

## 4. Quản Lý Chiều Thay Đổi Chậm (SCD Types) (Chương 5)

|**Loại SCD**|**Cơ chế**|**Ưu điểm**|**Nhược điểm / Đánh đổi**|
|---|---|---|---|
|**Type 0**|Giữ nguyên gốc (_Retain Original_)|Đơn giản, bất biến (vd: Ngày sinh)|Không cập nhật được dữ liệu sai|
|**Type 1**|Ghi đè (_Overwrite_)|Đơn giản, kích thước bảng không đổi|**Mất lịch sử hoàn toàn**, làm sai lệch báo cáo quá khứ|
|**Type 2**|Thêm dòng mới (_Add New Row_) + Key mới|**Chuẩn mực theo dõi lịch sử**; lưu lại chính xác quá khứ|Bảng phình to; cần quản lý `effective_date`, `expiration_date`, `is_current`|
|**Type 3**|Thêm cột mới (_Add New Attribute_)|So sánh trực tiếp giá trị cũ và mới trên 1 dòng|Chỉ lưu được 1 giá trị quá khứ gần nhất|
|**Type 4**|Tách Mini-Dimension|Xử lý tốt các thuộc tính thay đổi rất nhanh (vd: dải thu nhập)|Fact table phải chứa thêm FK trỏ tới mini-dimension|
|**Type 5**|Type 4 + Type 1 Outrigger|Query trạng thái hiện hành từ bảng khách hàng mà không cần join Fact|Cấu trúc ETL phức tạp hơn Type 4|
|**Type 6**|Lai: $2 + 3 + 1$ (Dòng Type 2 + Cột Current)|Báo cáo đồng thời trạng thái lịch sử (_as-was_) và hiện tại (_as-is_)|ETL phải update thuộc tính hiện hành trên toàn bộ các dòng cũ|
|**Type 7**|Fact chứa cả Surrogate Key (Type 2) và Durable Key (Type 1)|Rất linh hoạt; hỗ trợ cả 2 góc nhìn _as-was_ và _as-is_|Fact table tốn gấp đôi cột khóa ngoại cho cùng 1 dimension|

## 5. Tích Hợp Hệ Thống: Enterprise Bus Architecture (Chương 4 & 6)

### 5.1. Conformed Dimensions & Conformed Facts

- **Conformed Dimension (Chiều đồng nhất):** Một Dimension có cấu trúc và dữ liệu nhất quán được chia sẻ dùng chung giữa nhiều Fact table (vd: `Dim_Date`, `Dim_Product`).
    
    - **Shrunken Rollup:** Phiên bản thu gọn của Dimension chi tiết (vd: `Dim_Brand` là bản rollup của `Dim_Product`) dùng cho các Fact có mức hạt cao hơn (vd: Budget Fact).
        
- **Conformed Facts:** Đảm bảo cùng một chỉ số (vd: _Doanh thu thuần_) phải có cùng công thức tính và đơn vị đo trên mọi Fact table. Nếu khác công thức, bắt buộc phải đổi tên.
    

### 5.2. Drill-Across (Kỹ thuật kết hợp Fact-to-Fact)

- **Nguyên tắc vàng:** **Tuyệt đối không join trực tiếp Fact table với Fact table**. Thao tác này gây Cartesian product, trùng lặp số đo và làm sập hiệu năng.
    
- **Cơ chế Drill-Across (Multipass SQL):**
    
    1. Query riêng Fact A theo Conformed Dimension (Group by `Product_Key`).
        
    2. Query riêng Fact B theo Conformed Dimension (Group by `Product_Key`).
        
    3. Outer join / Merge kết quả của 2 query trên tầng BI hoặc SQL ngoài cùng dựa vào `Product_Key`.
        

```
[Query Fact_Sales]      → Group by Product_Key ↘
                                                  Full Outer Join on Product_Key → [Final Report]
[Query Fact_Inventory]  → Group by Product_Key ↗
```

## 6. Xử Lý Phân Cấp & Hành Vi Khách Hàng Phức Tạp (Chương 7 & 8)

### 6.1. Xử lý Cấu trúc Phân cấp (Hierarchies)

- **Fixed Depth (Độ sâu cố định):** Flatten toàn bộ thành các cột trong Dimension (vd: `Category → Subcategory → Product`). Tối ưu hiệu năng nhất.
    
- **Ragged / Variable Depth (Độ sâu không xác định - sơ đồ tổ chức công ty):**
    
    - Dùng **Hierarchy Bridge Table**: Bảng chứa mọi cặp `Ancestor - Descendant` kèm cột `Depth` và `Bottom_Flag`. Cho phép duyệt cây phân cấp bất kỳ độ sâu nào bằng SQL chuẩn.
        

### 6.2. Kỹ thuật CRM & Hành vi người dùng

- **Sparse Attributes (Thuộc tính thưa thớt):** Sử dụng Bridge table liên kết Khách hàng với danh sách `Name-Value pairs` thay vì tạo hàng trăm cột chứa toàn giá trị `NULL`.
    
- **Behavior Study Groups:** Lưu danh sách các `Customer_Key` thỏa mãn một điều kiện hành vi phức tạp vào một bảng tĩnh để tái sử dụng làm bộ lọc cho các query khác.
    
- **Step Dimension:** Chiều xác định số thứ tự bước trong chuỗi hành vi tuần tự (vd: Funnel checkout web: bước 1, bước 2, bước 3).
    

## 7. Các Sai Lầm Kiến Trúc Cần Tránh (Anti-Patterns)

1. **Snowflaking (Chuẩn hóa bảng chiều):** Tách Dimension thành nhiều bảng 3NF (tách Category khỏi Product). _Khắc phục:_ Luôn làm phẳng (flatten/denormalize) bảng chiều.
    
2. **Centipede Fact Table (Bảng sự kiện con rết):** Đưa hàng chục FK của các cấp phân cấp nhỏ lẻ vào Fact table khiến Fact có quá nhiều "chân". _Khắc phục:_ Gom các thuộc tính liên quan vào chung một Dimension lớn.
    
3. **Fact-to-Fact Join:** Join trực tiếp 2 bảng sự kiện. _Khắc phục:_ Dùng Conformed Dimension và kỹ thuật Drill-Across.
    
4. **Header / Line Item Fact Table:** Tạo 1 Fact cho Header và 1 Fact cho Line Item rồi join lại. _Khắc phục:_ Đưa Fact về mức hạt Line Item, phân bổ (allocate) các chi phí cấp Header xuống từng dòng Line Item.
    
5. **Dùng Natural Key làm PK bảng chiều:** Dùng mã nguồn từ ERP/CRM làm khóa chính. _Khắc phục:_ Luôn dùng Surrogate Key.
    
6. **Lưu trữ giá trị NULL ở Foreign Key của Fact:** Gây mất dòng dữ liệu khi `INNER JOIN`. _Khắc phục:_ Tạo dòng mặc định trong Dimension với Key `-1` (_Unknown / Not Applicable_).
    

## 8. Quy Trình Vòng Đời & ETL Subsystems (Chương 17, 19, 20)

### 8.1. Kimball Lifecycle & 3 Luồng Song Song

Sau khi hoàn thành **Định nghĩa Yêu cầu Nghiệp vụ** (Business Requirements), triển khai đồng thời 3 luồng:

1. **Technology Track:** Thiết kế kiến trúc hạ tầng → Chọn & cài đặt công cụ.
    
2. **Data Track:** Mô hình hóa đa chiều → Thiết kế vật lý → Xây dựng pipeline ETL.
    
3. **BI Applications Track:** Thiết kế & phát triển các báo cáo tiêu chuẩn (10–15 core reports/dashboards).
    

### 8.2. Kiến trúc 34 ETL Subsystems (4 Nhóm)

- **Extracting (3):** Đọc dữ liệu từ đa nguồn (RDBMS, Flat file, Streaming, ERP, APIs).
    
- **Cleaning & Conforming (5):** Bộ lọc chất lượng dữ liệu (Data quality screens), Error Event Schema (ghi log lỗi), Audit Dimension Assembler (gắn metadata chất lượng), Deduplication.
    
- **Delivering (13):** Surrogate Key Generator & Pipeline, SCD Manager, Fact & Bridge Builder, Late Arriving Data Handler (tìm lùi Surrogate Key lịch sử), Aggregate & Cube Builder.
    
- **Managing (13):** Job Scheduler, Backup/Recovery, Lineage & Dependency Tracking, Version Control.
    

### 8.3. Đánh đổi trong Real-time DW

- **Quy luật:** Tốc độ càng cao (Streaming/Micro-batch) thì chất lượng và mức độ làm sạch dữ liệu chuyên sâu càng giảm.
    
- **Giải pháp Phân vùng Real-time:**
    
    - Dữ liệu trong ngày đẩy vào **Hot Partition / In-memory Table** (không index, không aggregate phức tạp để ghi cực nhanh).
        
    - Chạy batch job cuối ngày để làm sạch, xử lý SCD, tính aggregate và **Merge** phân vùng nóng vào Fact table lịch sử cố định.
        

## 9. Big Data Analytics & Nguyên Tắc Kimball Hiện Đại (Chương 21)

Big Data không thay thế Dimensional Modeling mà mở rộng nó:

- **Tư duy đa chiều không đổi:** Người dùng cuối luôn phân tích theo _Context (Dimensions)_ và _Metrics (Facts)_, dù underlying storage là Data Lake, Lakehouse hay NoSQL.
    
- **Late Binding (Schema-on-Read):** Với dữ liệu phi cấu trúc/bán cấu trúc, cấu trúc chỉ được định hình khi truy vấn thay vì ép schema cứng ngay từ tầng lưu trữ thô.
    
- **Data Highway (Kiến trúc 5 bộ đệm dữ liệu):**
    
    $$\text{Raw Source} \longrightarrow \text{Real-Time Cache} \longrightarrow \text{Business Activity Cache} \longrightarrow \text{Top Line Cache} \longrightarrow \text{EDW / Data Lakehouse}$$
    
- **Backflow to Earlier Caches:** Các thuộc tính Dimension chuẩn hóa và Durable Keys đã được quản trị sạch sẽ trong DW cần được đẩy ngược (sync) về các hệ thống phục vụ tầng real-time để dữ liệu luồng cũng tận dụng được ngữ cảnh phân tích phong phú.
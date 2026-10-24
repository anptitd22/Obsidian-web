Trong SQL, con trỏ (cursor) là một đối tượng cơ sở dữ liệu được sử dụng để xử lý dữ liệu từng hàng một, hữu ích khi cần xử lý từng hàng thay vì xử lý hàng loạt. Nó lưu trữ dữ liệu tạm thời cho các thao tác như SELECT, UPDATE hoặc DELETE.

- Hữu ích để áp dụng logic tùy chỉnh, các phép toán có điều kiện hoặc cập nhật từng bước.

- Thường được sử dụng trong PL/SQL hoặc T-SQL cho các tác vụ như vòng lặp, logic điều kiện hoặc các thủ tục phức tạp.

## Vòng Đời Hoạt Động (5 Bước Tiêu Chuẩn)

Mọi hệ quản trị cơ sở dữ liệu (PostgreSQL, MySQL, SQL Server, Oracle) đều tuân theo chu trình 5 bước:

```
[1. DECLARE] ──► [2. OPEN] ──► [3. FETCH] (Vòng lặp) ──► [4. CLOSE] ──► [5. DEALLOCATE]
```

![[cusor_sql_1.png]]

1. **DECLARE (Khai báo):** Định nghĩa tên cursor và gán nó với câu lệnh `SELECT`.
    
2. **OPEN (Mở):** Thực thi câu lệnh `SELECT`, nạp tập kết quả vào bộ nhớ và đặt con trỏ trước hàng đầu tiên.
    
3. **FETCH (Lấy dòng):** Di chuyển con trỏ đến hàng tiếp theo và gán giá trị các cột vào các biến cục bộ. Quá trình này lặp lại cho đến khi hết dữ liệu (kiểm tra qua biến trạng thái như `@@FETCH_STATUS = 0` trong SQL Server hoặc `NOT FOUND` trong MySQL/Postgres).
    
4. **CLOSE (Đóng):** Đóng cursor và giải phóng tập kết quả đang mở (nhưng vẫn giữ lại định nghĩa cấu trúc).
    
5. **DEALLOCATE / FREE (Hủy bỏ):** Giải phóng hoàn toàn vùng nhớ và tài nguyên liên quan đến cursor.

## Use of Cursor

Nên sử dụng con trỏ chuột một cách cẩn thận vì chúng hữu ích trong các trường hợp như:

- Thực hiện logic điều kiện từng hàng một
- Lặp qua dữ liệu để tính toán hoặc chuyển đổi các trường.
- Lặp lại các tập kết quả để cập nhật hoặc biến đổi có điều kiện.
- Xử lý cấu trúc dữ liệu phân cấp hoặc đệ quy
- Thực hiện các tác vụ dọn dẹp không thể thực hiện bằng một truy vấn SQL duy nhất.

## Types of Cursors in SQL

SQL cung cấp hai loại con trỏ chính, mỗi loại phù hợp với các tình huống khác nhau và tùy thuộc vào mức độ kiểm soát mà chúng ta muốn.

### 1. Implicit Cursors

Trong PL/SQL, khi thực hiện các thao tác INSERT, UPDATE hoặc DELETE, một con trỏ ngầm định sẽ tự động được tạo ra. Con trỏ này lưu giữ dữ liệu cần chèn hoặc xác định các hàng cần cập nhật hoặc xóa. Chúng ta có thể gọi con trỏ này là con trỏ SQL trong mã của mình. Nó được quản lý hoàn toàn bởi công cụ SQL mà không cần khai báo rõ ràng.

**Useful Attributes**:

- ****%FOUND****:Đúng nếu thao tác SQL ảnh hưởng đến ít nhất một hàng.
- ****%NOTFOUND****: Đúng nếu không có hàng nào bị ảnh hưởng.
- ****%ROWCOUNT****: Trả về số lượng hàng bị ảnh hưởng
- ****%ISOPEN****:  Kiểm tra xem con trỏ có đang mở hay không.
#### Example: Using Implicit Cursor for Bulk Updates

Chương trình này cập nhật một bảng bằng cách tăng lương của mỗi nhân viên lên 1500. Sau khi cập nhật, thuộc tính SQL%ROWCOUNT được sử dụng để tìm ra số lượng hàng bị ảnh hưởng bởi thao tác này.

**Query:**

```
DECLARE    
   total_rows number;   
BEGIN   
   UPDATE Emp   
   SET Salary = Salary + 1500;   
   
   total_rows := SQL%ROWCOUNT;  
   
   dbms_output.put_line(total_rows || ' rows updated.');   
END;
```

**Output**

```
5 Emp selected    
PL/SQL procedure successfully completed.
```

**Explanation:**

- BULK_ROWCOUNT và %BULK_EXCEPTIONS được sử dụng với câu lệnh FORALL.
- Câu lệnh FORALL được sử dụng để thực hiện nhiều thao tác DML một cách hiệu quả.
- BULK_ROWCOUNT trả về số lượng hàng bị ảnh hưởng bởi mỗi thao tác DML.
- BULK_EXCEPTIONS trả về thông tin về các ngoại lệ xảy ra trong quá trình thực hiện các thao tác hàng loạt.

### 2. Explicit Cursors

Đây là các con trỏ do người dùng định nghĩa, được tạo ra một cách rõ ràng bởi người dùng cho các thao tác tùy chỉnh. Chúng cung cấp quyền kiểm soát hoàn toàn đối với mọi phần trong vòng đời của chúng: khai báo, mở, truy xuất, đóng và giải phóng. Chúng được sử dụng để truy xuất dữ liệu từng hàng một với quyền kiểm soát hoàn toàn vòng đời của con trỏ.

##### **Explicit cursors are useful when:**

- Chúng ta cần lặp qua các kết quả một cách thủ công.
- Mỗi hàng cần được xử lý bằng logic riêng.
- Chúng ta cần truy cập vào các thuộc tính của hàng trong quá trình xử lý.

#### Example: Using an Explicit Cursor

Dưới đây là một ví dụ hoàn chỉnh về việc khai báo, mở, truy xuất, đóng và giải phóng con trỏ. Quy trình này minh họa cách sử dụng con trỏ tường minh cho các thao tác từng hàng, bao gồm quản lý tài nguyên và truy xuất kết quả.

**Query:**

```
DECLARE emp_cursor CURSOR FOR SELECT Name, Salary FROM Employees;  
  
BEGIN  
    OPEN emp_cursor;  
  FETCH NEXT FROM emp_cursor INTO @Name, @Salary;  
   WHILE @@FETCH_STATUS = 0  
   BEGIN  
      PRINT 'Name: ' + @Name + ', Salary: ' + CAST(@Salary AS VARCHAR);  
      FETCH NEXT FROM emp_cursor INTO @Name, @Salary;  
   END;  
  
   CLOSE emp_cursor;  
  
   DEALLOCATE emp_cursor;  
END;
```

**Output**:

![[cusor_sql_2.png]]

| **Implicit Cursors**                                                                                 | **Explicit Cursors**                                                  |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **Được tạo tự động** bởi SQL engine                                                                  | **Được tạo thủ công** bởi người dùng/lập trình viên                   |
| **Không yêu cầu khai báo**                                                                           | **Yêu cầu phải khai báo** trước khi sử dụng                           |
| **Được quản lý tự động** bởi SQL                                                                     | **Được điều khiển thủ công** bằng các lệnh `OPEN`, `FETCH` và `CLOSE` |
| Thường dùng cho các **thao tác DML đơn giản** (`INSERT`, `UPDATE`, `DELETE`, `SELECT INTO` đơn dòng) | Dùng cho **xử lý logic phức tạp theo từng hàng (row-by-row)**         |
| **Ít khả năng kiểm soát** hơn                                                                        | **Cung cấp nhiều quyền kiểm soát** và tính linh hoạt hơn              |
| **Nhanh hơn và hiệu quả hơn**                                                                        | **Chậm hơn** do chi phí quản lý và xử lý thủ công                     |

## Advantages of Using Cursors

- **Row-by-row processing:** Hữu ích cho các thao tác chi tiết.
- **Iterative handling:** Có thể lặp qua dữ liệu nhiều lần.
- **Handles complex relationships:** Làm việc với dữ liệu phân cấp/phức tạp
- **Conditional operations:** Hỗ trợ cập nhật, xóa và chèn dữ liệu dựa trên điều kiện.
- **Flexible for complex cases: Hữu ích khi các truy vấn dựa trên tập hợp thất bại

## Limitations of Cursors

- **Slow performance:** Xử lý từng hàng một
- **High resource usage:** Tiêu tốn bộ nhớ và khóa dữ liệu
- **Complex syntax:** Yêu cầu nhiều bước (khai báo, mở, lấy dữ liệu, đóng).
- **Not suitable for large data:** Hiệu năng giảm khi xử lý tập dữ liệu lớn.
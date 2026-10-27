UDF là viết tắt của User-Defined Function (Hàm do người dùng định nghĩa).

UDF cho phép bạn định nghĩa logic tùy chỉnh mà Spark có thể áp dụng cho các cột của DataFrame.

Ví dụ:

```
name
----------------
Alice
Bob
Charlie
```

Chúng tôi muốn tính độ dài của mỗi tên thì Spark đã cung cấp sẵn:

```
length("name")
```

Nhưng hãy tưởng tượng rằng sự chuyển đổi đó là điều mà Spark không cung cấp sẵn.

Chúng ta có thể viết:

```python
def custom_logic(value):
    return ...
```

và chuyển đổi nó thành một hàm UDF của Spark.

Về mặt khái niệm:

```
DataFrame
   │
   ▼
Column
   │
   ▼
Python UDF
   │
   ▼
Custom Python Logic
   │
   ▼
Result Column
```

Hiện tại, PySpark hỗ trợ một số loại hàm UDF Python, bao gồm **scalar Python UDFs, Pandas UDFs, and Arrow UDFs**.






- Lưu cache giá trị chỉ đọc (read-only) trong bộ nhớ trên tất cả các node

- Khởi tạo bằng `sc.broadcast(v)` để phân phối một biến chỉ đọc (read-only) đến từng node làm việc thay vì sao chép theo từng task.
    - Trích xuất giá trị bằng phương thức `.value`.
    - Giải phóng tài nguyên bằng `.unpersist()` hoặc `.destroy()`.

```python
>>> broadcastVar = sc.broadcast([1, 2, 3])
<pyspark.core.broadcast.Broadcast object at 0x102789f10>

>>> broadcastVar.value
[1, 2, 3]
```


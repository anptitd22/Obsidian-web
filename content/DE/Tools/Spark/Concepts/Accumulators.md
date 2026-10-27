- Biến chỉ thực hiện phép cộng tích lũy (như biến đếm counter hoặc tổng sum) hay biến tích lũy chỉ hỗ trợ thao tác "cộng" (sum).

- Worker chỉ có thể thêm giá trị bằng `add()`, không thể đọc. Chỉ Driver program mới có thể đọc giá trị qua `.value`.

- Được bảo đảm cập nhật chính xác (chỉ áp dụng 1 lần cho mỗi task) khi thực hiện trong các **Action**.

![[accumulator.png]]

```python
>>> accum = sc.accumulator(0)
>>> accum
Accumulator<id=0, value=0>

>>> sc.parallelize([1, 2, 3, 4]).foreach(lambda x: accum.add(x))
...
10/09/29 INFO SparkContext: Tasks finished in 0.317106 s

>>> accum.value
10
```

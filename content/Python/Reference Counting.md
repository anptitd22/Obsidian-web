---
title: Reference Counting
tags:
  - Python
---

Với python, mọi số, string, ... được khởi tạo được coi là một object
Khi các biến trỏ tới object đó (hay vùng bộ nhớ đó) thì số reference count sẽ tăng lên
Nếu reference count = 0 thì gc sẽ tự động dọn dẹp để giải phóng Ram

```python
import sys

# Thử với một số nhỏ (được interning)
n = 5
print(sys.getrefcount(5))
# Kết quả có thể là 33

# Thử với một số rất lớn (không được interning)
x = 9876543215353535
print(sys.getrefcount(9876543215353535)) # Kết quả là 4
y = x
print(sys.getrefcount(9876543215353535)) # Số tham chiếu sẽ tăng lên rõ rệt là 5
```

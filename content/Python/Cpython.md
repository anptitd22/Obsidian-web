---
title: Cpython
tags:
  - Python
---

Phiên bản chuẩn của ngôn ngữ lập trình Python, được phân phối trên python.org. Thuật ngữ “CPython” được sử dụng khi cần thiết để phân biệt phiên bản này với các phiên bản khác như Jython hoặc IronPython.

## Quy trình compile code

Dịch sang **Bytecode của Python** rồi chạy trên một **Máy ảo (Virtual Machine)** được viết bằng C.

Bước 1: Từ `.py` sang Bytecode (`.pyc`)

Khi bạn chạy một file Python, trình thông dịch CPython sẽ đọc code của bạn và dịch nó sang một dạng mã trung gian gọi là **Bytecode**.

- **Đặc điểm:** Bytecode là những lệnh rất đơn giản mà con người khó đọc nhưng máy tính lại hiểu rất nhanh.
- **Lưu trữ:** Bạn thường thấy các file này trong thư mục `__pycache__` với đuôi `.pyc`.
- **Ví dụ:** Lệnh `a + b` trong Python sẽ được dịch thành lệnh bytecode kiểu như `BINARY_ADD`.

Bước 2: Máy ảo Python (Python Virtual Machine - PVM)

Đây là nơi ngôn ngữ C xuất hiện. PVM là một chương trình **được viết bằng C**.

- PVM sẽ đọc từng dòng Bytecode ở Bước 1.    
- Với mỗi lệnh Bytecode, PVM có sẵn các đoạn mã C tương ứng để thực hiện lệnh đó.
- **Ví dụ:** Khi thấy lệnh `BINARY_ADD`, PVM sẽ gọi một hàm C (như `PyNumber_Add`) để thực hiện phép cộng thực sự trên CPU.

Bước 3: Từ PVM sang Mã máy (Machine Code)

Cuối cùng, các hàm bằng C trong PVM sẽ được biên dịch sẵn thành **mã máy** (những con số 0 và 1) để CPU của bạn có thể hiểu và xử lý trực tiếp trên phần cứng.
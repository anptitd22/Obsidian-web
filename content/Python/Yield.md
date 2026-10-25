Khi một hàm có chứa từ khóa `yield`, nó trở thành **Generator Function**. Khi bạn gọi hàm này, nó **không chạy code bên trong ngay lập tức**, mà nó trả về một đối tượng gọi là **Generator**(đối tượng có thể lặp). 

**Tác dụng chính của `yield`**

- **Tiết kiệm bộ nhớ (RAM):** Do không cần lưu trữ toàn bộ chuỗi dữ liệu vào danh sách (list) ở bộ nhớ RAM, `yield` chỉ sinh ra phần tử tiếp theo khi được yêu cầu (Lazy Evaluation).
    
- **Xử lý chuỗi dữ liệu lớn hoặc vô tận:** Rất phù hợp khi đọc file dữ liệu vài GB, làm việc với database lớn, hoặc tạo chuỗi số vô hạn.
    
- **Tăng hiệu năng:** Giúp chương trình bắt đầu xử lý dữ liệu ngay lập tức mà không cần đợi tính toán xong toàn bộ danh sách.

```python
def tao_generator(n):
    for i in range(n):
        yield i  # Trả về i và tạm dừng tại đây


gen = tao_generator(1000000)  # Gần như không tốn RAM

# Lấy từng giá trị bằng vòng lặp for
for num in tao_generator(3):
    print(num)
# Đầu ra:
# 0
# 1
# 2
```

**Khi nào nên dùng `yield`?**

- Khi làm việc với **tập dữ liệu cực lớn** (ngàn dòng log, file CSV hàng GB) không thể hoặc không nên load hết vào RAM.
    
- Khi bạn **chỉ cần duyệt qua dữ liệu một lần** bằng vòng lặp `for`.
    
- Khi tạo các hàm cần tính toán chuỗi dữ liệu phức tạp hoặc **chuỗi vô hạn** (ví dụ: tạo dãy số Fibonacci không giới hạn).
    

**Khi nào KHÔNG nên dùng `yield`?**

- Khi bạn cần truy cập ngẫu nhiên các phần tử theo chỉ số (như `data[5]`).
    
- Khi bạn cần duyệt đi duyệt lại cùng một tập dữ liệu nhiều lần.

### next()

- **Lần gọi đầu tiên (`next()`):** Hàm chạy từ đầu đến khi gặp dòng `yield` đầu tiên. Nó trả về giá trị sau `yield` và dừng lại.

- **Lần gọi tiếp theo:** Hàm chạy tiếp từ nơi nó vừa dừng. Toàn bộ biến cục bộ từ lần trước vẫn còn nguyên.

- **Khi hết code:** Nó sẽ tung ra lỗi `StopIteration` để báo rằng "tôi hết hàng rồi".

### send()

Đóng vai trò vừa lấy dữ liệu từ yield và đẩy dữ liệu từ phương thức send() vào lại.

Ví dụ:

```python
def may_tinh_tong():
    tong = 0
    print("--- Máy tính tổng đã sẵn sàng! ---")
    while True:
        # yield nhả ra 'tong' hiện tại và ĐỢI nhận số mới vào 'so_moi'
        so_moi = yield tong 
        
        if so_moi is None:
            break # Thoát nếu không gửi gì
            
        tong += so_moi
        print(f"-> Đã nhận: {so_moi}, Tổng mới: {tong}")

# 1. Khởi tạo generator
gen = may_tinh_tong()

# 2. Bước quan trọng: 'Mồi' (Prime) generator
# Bạn phải gọi next() hoặc .send(None) lần đầu để hàm chạy đến dòng yield đầu tiên
dau_tien = next(gen) 
print(f"Giá trị ban đầu: {dau_tien}")

# 3. Bắt đầu gửi dữ liệu vào
print(f"Kết quả trả về từ send: {gen.send(10)}") # Gửi 10, nhận về 10
print(f"Kết quả trả về từ send: {gen.send(20)}") # Gửi 20, nhận về 30
print(f"Kết quả trả về từ send: {gen.send(5)}")  # Gửi 5, nhận về 35
```
- Lần 1: next(gen) -> yield -> Lúc này tong = 0 và đóng băng
- Lần 2: gen.send(10) -> release -> biến ở yield: so moi = 10 -> tong = 10 -> yield -> trả ra tong = 10 -> đóng băng
- Lần 3: ...
- Lần 4: ...

### yield from

`yield from <iterable>` cho phép bạn nhả ra toàn bộ giá trị từ một danh sách hoặc một generator khác mà không cần viết vòng lặp `for`.

```python
def kho_hang():
    yield "Táo"
    yield "Lê"

def dai_ly():
    yield "Kẹo"
    yield from kho_hang() # Nhả trực tiếp Táo và Lê từ kho_hang
    yield "Bánh"
```


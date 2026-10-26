---
title: Async
tags:
  - Python
---

Asyncio là một thư viện dùng để viết mã xử lý **concurrent** bằng cú pháp async/await.

Asyncio được sử dụng làm nền tảng cho nhiều framework bất đồng bộ Python, cung cấp các máy chủ web và mạng hiệu năng cao, thư viện kết nối cơ sở dữ liệu, hàng đợi tác vụ phân tán, v.v.

```python
import asyncio

async def main():
    print('Hello ...')
    await asyncio.sleep(1)
    print('... World!')

asyncio.run(main())
```

Asyncio thường là lựa chọn hoàn hảo cho IO-bound và high-level **structured** network code

High-level APIs (cần nhớ)

- Coroutines and tasks
- Queues
- Exceptions
- Runners
- Synchronization Primitives

Low-level APIs (cần nhớ)

- Event loop
- Futures

### Event Loop

Được ví như "nhạc trưởng" điều phối mọi thứ. Vòng lặp chứa một danh sách các công việc (jobs) cần chạy, nó sẽ lần lượt lấy từng công việc ra và cấp quyền điều khiển cho công việc đó. Khi công việc tạm dừng hoặc chạy xong, quyền điều khiển được trả lại cho vòng lặp để tiếp tục với công việc khác. Để hệ thống hiệu quả, các công việc phải hoạt động hợp tác và không được chiếm dụng tài nguyên quá lâu. Nếu không còn tác vụ nào đang chờ thực thi, vòng lặp sự kiện đủ thông minh để nghỉ ngơi và tránh lãng phí chu kỳ CPU một cách không cần thiết, và sẽ quay lại khi có thêm việc cần làm.

Việc thực thi hiệu quả phụ thuộc vào việc các công việc chia sẻ và hợp tác tốt với nhau; một công việc 'tham lam' có thể chiếm quyền điều khiển và khiến các công việc khác bị 'bỏ đói', làm cho phương pháp vòng lặp sự kiện tổng thể trở nên khá vô dụng.

```python
import asyncio

# This creates an event loop and indefinitely cycles through
# its collection of jobs.
event_loop = asyncio.new_event_loop()
event_loop.run_forever()
```

### Asynchronous functions and coroutines

Từ khóa [async def](https://docs.python.org/3/reference/compound_stmts.html#async-def), khác với từ khóa `def` thông thường, biến nó thành một hàm bất đồng bộ (hay còn gọi là "coroutine function"). Việc gọi từ khóa này sẽ tạo ra và trả về một [coroutine](https://docs.python.org/3/library/asyncio-task.html#coroutine) object.

```python
async def loudmouth_penguin(magic_number: int):
    print(
     "I am a super special talking penguin. Far cooler than that printer. "
     f"By the way, my lucky number is: {magic_number}."
    )
```

Việc gọi hàm bất đồng bộ `loudmouth_penguin` không thực thi câu lệnh print; thay vào đó, nó tạo ra một coroutine object:

```
loudmouth_penguin(magic_number=3)
<coroutine object loudmouth_penguin at 0x104ed2740>
```

>[!note]
>Các thuật ngữ “coroutine function” và “coroutine object” thường bị nhầm lẫn với nhau thành coroutine. Điều đó có thể gây nhầm lẫn! coroutine function là đại diện cho hàm async def tĩnh còn coroutine object là khi hàm async được thực thi. Nói cách khác coroutine function chạy sẽ tạo ra coroutine ọbject. Trong bài viết này, coroutine được định nghĩa là coroutine object

Coroutine đại diện cho phần thân hoặc logic của hàm. Một coroutine phải được khởi tạo một cách rõ ràng; nói cách khác, chỉ tạo ra coroutine thôi thì chưa đủ để khởi động nó. Đặc biệt, coroutine có thể được tạm dừng và tiếp tục tại nhiều điểm khác nhau trong phần thân của hàm. Khả năng tạm dừng và tiếp tục đó chính là điều cho phép hoạt động bất đồng bộ!

Các coroutine và coroutine function được xây dựng bằng cách tận dụng chức năng của các [generators](https://docs.python.org/3/glossary.html#term-generator-iterator) và [generator functions](https://docs.python.org/3/glossary.html#term-generator). Hãy nhớ lại, generator function là một hàm trả về [`yield`](https://docs.python.org/3/reference/simple_stmts.html#yield), giống như hàm này:

```python
def get_random_number():
    # This would be a bad random number generator!
    print("Hi")
    yield 1
    print("Hello")
    yield 7
    print("Howdy")
    yield 4
    ...
```

Tương tự như coroutine function, việc gọi một generator function không thực sự chạy hàm đó. Thay vào đó, nó tạo ra một generator object:

```console
get_random_number()
<generator object get_random_number at 0x1048671c0>
```

Bạn có thể chuyển sang kết quả tiếp theo của trình tạo bằng cách sử dụng hàm tích hợp `next()`. Nói cách khác, trình tạo chạy, sau đó tạm dừng. Ví dụ:

```python
generator = get_random_number()
next(generator)
Hi
1
next(generator)
Hello
7
```

### Tasks

Nói một cách đơn giản, các [tasks](https://docs.python.org/3/library/asyncio-task.html#asyncio-task-obj) là các coroutine (không phải hàm coroutine) được gắn với một event loop. Một task cũng duy trì một danh sách các hàm gọi lại (callback), tầm quan trọng của chúng sẽ được làm rõ trong chốc lát khi chúng ta thảo luận về [`await`](https://docs.python.org/3/reference/expressions.html#await). Cách được khuyến nghị để tạo task là thông qua [`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task "asyncio.create_task").

Việc create_task sẽ tự động lên lịch thực thi cho task đó (bằng cách thêm một hàm gọi (call back) lại để chạy nó trong danh sách việc cần làm của event loop, tức là tập hợp các công việc).

Asyncio tự động liên kết các task với event loop cho bạn. Việc liên kết tự động này được thiết kế có chủ đích trong asyncio vì mục đích đơn giản hóa. Nếu không có nó, bạn sẽ phải theo dõi đối tượng event loop và truyền nó cho bất kỳ coroutine function nào muốn tạo task, làm tăng thêm sự rườm rà không cần thiết cho mã của bạn.

```python
coroutine = loudmouth_penguin(magic_number=5)
# This creates a Task object and schedules its execution via the event loop.
task = asyncio.create_task(coroutine)
```

```python
async def main():
    # Cách A: Dùng create_task
    task = asyncio.create_task(download(1))
    await asyncio.sleep(0.5) # Trong 0.5 giây này, task đã chạy được một nửa!
    await task # Chỉ cần đợi thêm 0.5 giây nữa là xong.

    # Cách B: Dùng Coroutine trực tiếp
    coro = download(2)
    await asyncio.sleep(0.5) # Trong 0.5 giây này, coro chả làm gì cả, nó chỉ là object nằm yên.
    await coro # Bây giờ mới bắt đầu chạy, phải đợi đủ 1 giây nữa.
```
### await

`await` là một từ khóa trong Python thường được sử dụng theo một trong hai cách khác nhau:

```python
await task
await coroutine
```

Về cơ bản, hành vi của await phụ thuộc vào loại đối tượng đang được chờ đợi.

Việc chờ đợi một task sẽ chuyển quyền điều khiển từ task hoặc coroutine hiện tại sang event loop. Trong quá trình chuyển giao quyền điều khiển, một vài điều quan trọng sẽ xảy ra. Chúng ta sẽ sử dụng ví dụ mã sau để minh họa:

```python
async def plant_a_tree():
    dig_the_hole_task = asyncio.create_task(dig_the_hole())
    await dig_the_hole_task

    # Other instructions associated with planting a tree.
    ...
```

1. Gặp dòng `asyncio.create_task(...)`:

- Hàm `dig_the_hole()` được gói thành một **Task** và lập tức đưa vào hàng đợi của Event Loop.
    
- Lúc này task đào hố đã sẵn sàng để chạy, nhưng chưa chạy ngay vì người quản lý (Event Loop) vẫn đang để CPU cho hàm `plant_a_tree()` chạy tiếp.
    
2. Gặp dòng `await dig_the_hole_task`:

Đây là điểm mấu chốt. Lệnh `await` ở đây làm 3 việc:

- **Tạm dừng:** Hàm `plant_a_tree()` tự dừng lại (đóng băng tại dòng `await`, chưa chạy các lệnh phía sau).
    
- **Đăng ký việc cần làm tiếp theo (Callback):** Nó dán một mảnh giấy ghi chú vào task đào hố: _"Này `dig_the_hole_task`, chừng nào làm xong thì nhớ đánh thức `plant_a_tree()` dậy để làm tiếp nhé"_.
    
- **Nhường quyền:** `plant_a_tree()` trả lại quyền điều khiển cho Event Loop.
    
3. Event Loop làm việc:

- Người quản lý thấy `plant_a_tree()` đang ngủ chờ việc, liền chuyển lượt chạy cho các task khác đang đợi trong hàng đợi, trong đó có `dig_the_hole_task`.
    
- `dig_the_hole_task` bắt đầu đào hố. Trong lúc đào hố nếu nó cũng phải chờ (ví dụ chờ I/O, `asyncio.sleep`), nó lại nhường quyền cho các tác vụ khác.
    
4. Khi `dig_the_hole_task` làm xong:

- Task đào hố hoàn thành, nó kiểm tra mảnh giấy ghi chú đã dán ở Bước 2.
    
- Nó báo với Event Loop: _"Tôi xong rồi, hãy đưa `plant_a_tree()` trở lại danh sách sẵn sàng chạy"_.
    
- Event Loop kích hoạt lại `plant_a_tree()`, tiếp tục chạy từ dòng ngay sau chữ `await` (lấp đất, tưới nước...).

Không giống như các task, việc chờ đợi một coroutine không trả lại quyền điều khiển cho task! Việc gói một coroutine trong một task trước, rồi sau đó chờ đợi task đó sẽ dẫn đến việc trả lại quyền điều khiển. Hành vi của việc chờ đợi coroutine về cơ bản giống như việc gọi một hàm Python thông thường, đồng bộ. Hãy xem xét chương trình này:

```python
import asyncio

async def coro_a():
   print("I am coro_a(). Hi!")

async def coro_b():
   print("I am coro_b(). I sure hope no one hogs the event loop...")

async def main():
   task_b = asyncio.create_task(coro_b())
   num_repeats = 3
   for _ in range(num_repeats):
      await coro_a()
   await task_b

asyncio.run(main())
```

Câu lệnh đầu tiên trong coroutine `main()` tạo ra `task_b` và lên lịch thực thi nó thông qua event loop. Sau đó, `coro_a()` được chờ đợi liên tục. Quyền điều khiển không bao giờ được chuyển giao cho event loop, đó là lý do tại sao chúng ta thấy đầu ra của cả ba lần gọi `coro_a()` trước khi đầu ra của `coro_b()` được thực thi:

```
I am coro_a(). Hi!
I am coro_a(). Hi!
I am coro_a(). Hi!
I am coro_b(). I sure hope no one hogs the event loop...
```

Nếu ta thay đổi `await coro_a()` thành `await asyncio.create_task(coro_a())`, hành vi sẽ thay đổi. Hàm `main()` của coroutine sẽ nhường quyền điều khiển cho event loop với câu lệnh đó. Event loop sau đó sẽ tiếp tục xử lý các công việc đang chờ xử lý, gọi `task_b` và sau đó là task bao bọc `coro_a()` trước khi tiếp tục hàm `main()` của coroutine.

```
I am coro_b(). I sure hope no one hogs the event loop...
I am coro_a(). Hi!
I am coro_a(). Hi!
I am coro_a(). Hi!
```

Hành vi của coroutine `await` có thể gây nhầm lẫn cho rất nhiều người! Ví dụ đó cho thấy việc chỉ sử dụng coroutine `await` có thể vô tình chiếm quyền điều khiển từ các task khác và làm đình trệ event loop. `asyncio.run()` có thể giúp bạn phát hiện những trường hợp như vậy thông qua cờ `debug=True`, cho phép [debug mode](https://docs.python.org/3/library/asyncio-dev.html#asyncio-debug-mode). Trong số những thứ khác, nó sẽ ghi lại bất kỳ coroutine nào chiếm quyền thực thi trong 100ms trở lên.

Thiết kế này cố tình đánh đổi một phần sự rõ ràng về mặt khái niệm xung quanh việc sử dụng await để cải thiện hiệu suất. Mỗi khi một task được chờ đợi, quyền điều khiển cần được truyền lên toàn bộ callstack đến event loop. Điều đó nghe có vẻ nhỏ nhặt, nhưng trong một chương trình lớn với nhiều câu lệnh await và callstack sâu, chi phí phát sinh đó có thể tích lũy thành một sự suy giảm hiệu suất đáng kể.

>[!note]
>Vậy await lồng nhau trong task nó sẽ là event loop + call stack

### Futures

[future](https://docs.python.org/3/library/asyncio-future.html#asyncio-future-obj)là một đối tượng được dùng để biểu thị trạng thái và kết quả của một phép tính. Thuật ngữ này ám chỉ ý tưởng về điều gì đó vẫn chưa xảy ra hoặc chưa diễn ra, và đối tượng này là một cách để theo dõi điều đó.

Một future có một vài thuộc tính quan trọng. Một trong số đó là trạng thái của nó, có thể là “pending”, “cancelled”, hoặc “done”. Một thuộc tính khác là kết quả của nó, được thiết lập khi trạng thái chuyển sang "đã hoàn thành". Không giống như coroutine, future không đại diện cho phép tính thực tế cần thực hiện; thay vào đó, nó đại diện cho trạng thái và kết quả của phép tính đó, giống như một đèn báo trạng thái (đỏ, vàng hoặc xanh lá cây) hoặc chỉ báo.

Lớp [`asyncio.Task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task "asyncio.Task") kế thừa từ [`asyncio.Future`](https://docs.python.org/3/library/asyncio-future.html#asyncio.Future "asyncio.Future") để có được các khả năng khác nhau này. Phần trước nói rằng các task lưu trữ một danh sách các hàm gọi lại, điều này không hoàn toàn chính xác. Trên thực tế, chính lớp `Future` mới là lớp thực hiện logic này, và `Task` kế thừa từ nó.

Bạn cũng có thể sử dụng Futures trực tiếp (không thông qua Tasks). Tasks tự đánh dấu là hoàn thành khi coroutine của nó kết thúc. Futures linh hoạt hơn nhiều và sẽ được đánh dấu là hoàn thành khi bạn yêu cầu. Theo cách này, chúng là giao diện linh hoạt cho phép bạn tự đặt ra các điều kiện chờ đợi và tiếp tục.

### A homemade asyncio.sleep

`asyncio.sleep(delay)` là hàm coroutine dùng để **tạm dừng tác vụ hiện tại** trong một khoảng thời gian nhất định mà **không làm nghẽn (block) toàn bộ chương trình**.

```python
await asyncio.sleep(3)
```

Bước 1: Tạo Future và đăng ký bộ hẹn giờ (Timer)

- `asyncio.sleep()` tạo ra một đối tượng `Future` trống (chưa có kết quả).
    
- Nó gọi hàm nội bộ của Event Loop (thường là `loop.call_later(3, future.set_result, None)`).
    
- Event Loop lưu bộ hẹn giờ này vào một cấu trúc dữ liệu hàng đợi ưu tiên (thường là Min-Heap) được sắp xếp theo mốc thời gian hết hạn (`thời_gian_hiện_tại + 3s`).
    
Bước 2: Tạm dừng coroutine và nhường quyền (Yield control)

- Lệnh `await` nhận diện rằng `Future` chưa hoàn thành, lập tức **đóng băng (suspend)** hàm hiện tại tại vị trí đó.
    
- Quyền điều khiển CPU lập tức được trả về cho **Event Loop**.
    
Bước 3: Event Loop xử lý việc khác

- Trong 3 giây đó, Event Loop không hề "ngủ". Nó tiếp tục quét hàng đợi để chạy các tác vụ tính toán khác hoặc chờ I/O mạng/database.
    
- Nếu không có việc gì làm, Event Loop sẽ ngủ ở mức OS kernel (dùng `epoll`, `kqueue` hoặc `select`) đúng bằng thời gian còn lại đến khi bộ hẹn giờ rung chuông.
    
Bước 4: Hết thời gian và đánh thức

- Khi đồng hồ chạm mốc 3 giây, Event Loop kích hoạt hàm callback: gọi `future.set_result(None)`.
    
- Trạng thái `Future` chuyển sang hoàn thành (done). Coroutine ban đầu được đưa vào hàng đợi sẵn sàng chạy (**Ready Queue**).
    
- Khi đến lượt, Event Loop khôi phục lại trạng thái hàm và chạy tiếp các dòng code ngay sau `await asyncio.sleep(3)`.

>[!note]
>`await asyncio.sleep(0)`: Không hoãn thời gian, mà dùng để **chủ động nhường CPU** (Yield to other tasks). Lúc này tác vụ tự đưa mình về cuối hàng đợi, cho phép Event Loop xử lý các task khác đang chờ trước khi quay lại hoàn thành nó.

Ví dụ:

```python
async def other_work():
    print("I like work. Work work.")

async def main():
    # Add a few other tasks to the event loop, so there's something
    # to do while asynchronously sleeping.
    work_tasks = [
        asyncio.create_task(other_work()),
        asyncio.create_task(other_work()),
        asyncio.create_task(other_work())
    ]
    print(
        "Beginning asynchronous sleep at time: "
        f"{datetime.datetime.now().strftime("%H:%M:%S")}."
    )
    await asyncio.create_task(async_sleep(3))
    print(
        "Done asynchronous sleep at time: "
        f"{datetime.datetime.now().strftime("%H:%M:%S")}."
    )
    # asyncio.gather effectively awaits each task in the collection.
    await asyncio.gather(*work_tasks)
```

Đây là toàn bộ kết quả đầu ra của chương trình:

```
$ python custom-async-sleep.py
Beginning asynchronous sleep at time: 14:52:22.
I like work. Work work.
I like work. Work work.
I like work. Work work.
Done asynchronous sleep at time: 14:52:25.
```

### Gather

Là tập hợp `await` , có thể await lần lượt hoặc await đồng thời nhiều task. Ngoài ra, tự động tạo task.

`create_task` kích hoạt việc chạy ngay lập tức, còn `gather` (khi nhận coroutine) sẽ kích hoạt việc chạy tại dòng lệnh đó.

```python
tasks = [download(i) for i in range(5)] # (1)
await asyncio.gather(*tasks)           # (2)
```

- **Tại (1):** Bạn chỉ mới tạo ra một danh sách các "bản thiết kế" (Coroutine objects). **Chưa có code nào được chạy cả.**
- **Tại (2):** Khi hàm `gather` được gọi, nó mới nhìn vào danh sách này, tự mình gọi `create_task` cho từng cái, và bắt đầu chạy đồng thời tất cả.

```python
tasks = [asyncio.create_task(download(i)) for i in range(5)] # (1)
await asyncio.gather(*tasks)                                # (2)
```

- **Tại (1):** Ngay khi vòng lặp chạy đến `create_task`, các task này đã được **đăng ký vào Event Loop và bắt đầu chạy ngay lập tức** ở hậu trường (ngay cả khi bạn chưa gọi `await`).
- **Tại (2):** Dòng `gather` lúc này chỉ đóng vai trò là một "điểm tập kết", đứng đợi cho các task đã chạy từ trước đó hoàn thành.

Hai đặc tính quan trọng của `gather`

```python
import asyncio

async def fetch_user(user_id):
    await asyncio.sleep(1)
    return {"id": user_id, "name": f"User_{user_id}"}

async def fetch_orders(user_id):
    await asyncio.sleep(2)
    return [101, 102, 103]

async def main():
    # gather tự bọc các coroutine thành Task và theo dõi
    user, orders = await asyncio.gather(
        fetch_user(1),
        fetch_orders(1)
    )
    print("User:", user)
    print("Orders:", orders)

asyncio.run(main())
```

- **Bảo toàn thứ tự kết quả:** Dù `fetch_user` hay `fetch_orders` cái nào xong trước, danh sách kết quả trả về luôn đúng theo thứ tự các tham số bạn truyền vào `gather()`.
    
- **Xử lý ngoại lệ (`return_exceptions`):**
    
    - **Mặc định (`return_exceptions=False`):** Nếu một tác vụ bị lỗi (ném exception), `gather` sẽ lập tức ném exception đó ra ngay, nhưng các tác vụ còn lại vẫn tiếp tục chạy ngầm trong nền.
        
    - **Khi bật (`return_exceptions=True`):** Nếu có lỗi, lỗi đó được coi như một giá trị trả về bình thường nằm trong danh sách kết quả thay vì làm sập chương trình.

### Runner

`asyncio.run(...)` thực chất chỉ là một hàm viết tắt tiện lợi bên ngoài của `asyncio.Runner` (mở runner, chạy đúng 1 lần rồi đóng).

Khi cần chạy nhiều coroutine nối tiếp nhau trên cùng một môi trường loop hoặc cần tùy biến cách khởi tạo loop, hãy dùng `asyncio.Runner`.

`Runner` hoạt động như một bộ khung bao bọc (context manager). Nó tạo ra một Event Loop duy nhất và giữ vòng lặp đó sống xuyên suốt khối lệnh `with`:

```python
import asyncio

async def task_one():
    await asyncio.sleep(1)
    print("Xong task 1")
    return "Data 1"

async def task_two():
    await asyncio.sleep(1)
    print("Xong task 2")
    return "Data 2"

# Sử dụng Runner làm Context Manager
with asyncio.Runner() as runner:
    # Cả hai tác vụ này chạy trên CÙNG MỘT event loop
    res1 = runner.run(task_one())
    res2 = runner.run(task_two())
    print(res1, res2)

# Ra khỏi khối with: Event loop và thread executor mới tự động đóng và dọn dẹp sạch sẽ
```

Tính năng chính:

- **`runner.run(coroutine)`:** Chạy coroutine trên event loop hiện tại của runner và trả về kết quả.
    
- **`runner.close()`:** Đóng vòng lặp và dọn dẹp (nếu dùng `with`, thao tác này được tự động gọi khi thoát khối lệnh).
    
- **`runner.get_loop()`:** Lấy đối tượng `loop` bên trong ra nếu cần can thiệp ở tầng thấp (low-level).

## Queue

Nó được thiết kế theo nguyên lý **FIFO** (First In, First Out – vào trước ra trước) và là công cụ tiêu chuẩn để triển khai mô hình kinh điển **Producer - Consumer** (Người tạo việc – Người xử lý việc).


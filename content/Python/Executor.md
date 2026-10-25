Khi trong luồng async có task CPU bound, một hàm def hay coroutine sẽ gây chặn event loop lúc này cần đưa các task, hàm đó vào các multi threading, được gọi là executor.

`Executor.submit()` sẽ tạo ra future object

Trên thực tế khi thiết kế app dùng fast api (async) có thể kết hợp với executor để có thể handle được nhiều task CPU bound (nhưng không tăng tốc độ xử lí tổng thể do [[GIL]]) 

## ThreadPoolExecutor

ThreadPoolExecutor là một lớp con của Executor sử dụng một pool of threads để thực thi các lệnh gọi bất đồng bộ.

```python
import time
def wait_on_b():
    time.sleep(5)
    print(b.result())  # b will never complete because it is waiting on a.
    return 5

def wait_on_a():
    time.sleep(5)
    print(a.result())  # a will never complete because it is waiting on b.
    return 6

executor = ThreadPoolExecutor(max_workers=2)
a = executor.submit(wait_on_b)
b = executor.submit(wait_on_a)
```

Và:

```python
def wait_on_future():
    f = executor.submit(pow, 5, 2)
    # This will never complete because there is only one worker thread and
    # it is executing this function.
    print(f.result())

executor = ThreadPoolExecutor(max_workers=1)
executor.submit(wait_on_future)
```

Config:

```
_class_ concurrent.futures.ThreadPoolExecutor(_max_workers=None_, _thread_name_prefix=''_, _initializer=None_, _initargs=()_)
```

>[!note]
>ThreadPoolExecutor không được khuyến khích cho việc chạy các task dài

### ThreadPoolExecutor Example

```python
import concurrent.futures
import urllib.request

URLS = ['http://www.foxnews.com/',
        'http://www.cnn.com/',
        'http://europe.wsj.com/',
        'http://www.bbc.co.uk/',
        'http://nonexistant-subdomain.python.org/']

# Retrieve a single page and report the URL and contents
def load_url(url, timeout):
    with urllib.request.urlopen(url, timeout=timeout) as conn:
        return conn.read()

# We can use a with statement to ensure threads are cleaned up promptly
with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
    # Start the load operations and mark each future with its URL
    future_to_url = {executor.submit(load_url, url, 60): url for url in URLS}
    for future in concurrent.futures.as_completed(future_to_url):
        url = future_to_url[future]
        try:
            data = future.result()
        except Exception as exc:
            print('%r generated an exception: %s' % (url, exc))
        else:
            print('%r page is %d bytes' % (url, len(data)))
```

## ProcessPoolExecutor

Đẩy 1 hàm def, coroutine hay task CPU bound gây chặn eventloop sang một process khác cho phép vượt qua giới hạn về [[GIL]]  

```
_class_ concurrent.futures.ProcessPoolExecutor(_max_workers=None_, _mp_context=None_, _initializer=None_, _initargs=()_, _max_tasks_per_child=None_)
```

### ProcessPoolExecutor Example

```python
import concurrent.futures
import math

PRIMES = [
    112272535095293,
    112582705942171,
    112272535095293,
    115280095190773,
    115797848077099,
    1099726899285419]

def is_prime(n):
    if n < 2:
        return False
    if n == 2:
        return True
    if n % 2 == 0:
        return False

    sqrt_n = int(math.floor(math.sqrt(n)))
    for i in range(3, sqrt_n + 1, 2):
        if n % i == 0:
            return False
    return True

def main():
    with concurrent.futures.ProcessPoolExecutor() as executor:
        for number, prime in zip(PRIMES, executor.map(is_prime, PRIMES)):
            print('%d is prime: %s' % (number, prime))

if __name__ == '__main__':
    main()
```


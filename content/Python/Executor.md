Khi trong luồng async có task CPU bound thì sẽ gây chặn event loop lúc này cần đưa các task đó vào các multi threading, được gọi là executor.

Trên thực tế khi thiết kế app dùng fast api (async) có thể kết hợp với executor để có thể handle được nhiều task CPU bound (nhưng không tăng tốc độ xử lí tổng thể do [[GIL]]) 
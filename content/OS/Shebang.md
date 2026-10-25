
Shebang là command line cho phép đưa thư mục interpreter thẳng vào file

```
#![đường_dẫn_trình_thông_dịch] [tùy_chọn]
```

Ví dụ:

```
#!/bin/bash 

echo "Chạy bằng Bash shell"
```

```python
#!/usr/bin/python3 

print("Hello from Python 3")
```

- `bash file.sh` = `#!bin/bash` trong `file.sh`

- `python3 file.py` = `#!/usr/bin/python3` trong `file.py`

### Cách thức hoạt động

Cấp quyền `chmod +x ./file.sh` và chạy `./file.sh`

>[!note]
>`#!` phải bắt đầu ở byte số 0 (cột 1, dòng 1). Không được có dấu cách hay dòng trống phía trước.


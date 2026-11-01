Nó sẽ lưu mặc định lưu các metadata schema của các connector trên 1 topic, topic này mặc định sẽ giữ dữ liệu trong 1 tuần và lưu trên RAM
Khi nó chạy sẽ sử dụng metadata từ RAM lên nhưng nếu kafka registry mà die thì sẽ mất hết dữ liệu
Giải pháp:
- Gọi API để lấy các metadata ra
- Sửa lại config lưu trên postgresql hoặc config topic giữ dữ liệu vĩnh viễn
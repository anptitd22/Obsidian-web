## CPU bound

Một tác vụ được gọi là CPU Bound khi thời gian hoàn thành công việc phụ thuộc hoàn toàn vào tốc độ của bộ vi xử lý - giới hạn task sử dụng CPU

## I/O bound

Một tác vụ được gọi là I/O Bound khi CPU đã xử lý xong việc của nó rất nhanh nhưng phải ngồi chờ các thành phần khác (ổ cứng, mạng) gửi hoặc nhận dữ liệu - giới hạn task sử dụng I/O

###  Tại sạo I/O bound lại không dùng CPU trong quá trình vận chuyển/xử lý ở giữa ?

CPU sinh ra thiên về xử lý tính toán phức tạp mà nó chỉ có thể giao tiếp với RAM

Vì nó sử dụng DMA Controller (Direct Memory Access - Truy cập bộ nhớ trực tiếp) có một vi mạch chuyên biệt sao chép lại chức năng quản lý bộ nhớ của CPU. Nó có các thanh ghi (registers) riêng để tự thực hiện các phép toán đơn giản:Con trỏ địa chỉ và Bộ đếm (PC)

DMA Process: Step to step

- Initialization: CPU cấu hình DMA cho nó biết dữ liệu nằm ở đâu và được chuyển đến đâu    
- CPU release: CPU giải phóng để thực hiện tác vụ khác
- Transfer: DMA sẽ tiếp quản các bus, data flow giữa thiết bị ngoại vi và ram, và truyền dữ liệu
- Interrupt: Khi hoàn thành, DMA sẽ gửi tín hiệu ngắt đến CPU để thông báo rằng job "DONE"
    
Bus Arbitration

- Chỉ có một master hoạt động tại một thời điểm
- Khi DMA muốn truyền dữ liệu nó sẽ HOLD request đến CPU
- CPU sẽ hoàn thành chu kì hiện tại và ngắt kết nối khỏi bus và gửi HLDA(Hold Ack), về cơ bản là giao quyền điều khiển cho DMA
    
DMA: có 3 cơ chế transfer

- Burst Mode
- Cycle Stealing
- Demand Mode


## 1. Tổng quan & Đặc tính nổi bật (Kafka Streams Highlights)

- **Khái niệm:** Thư viện client nhẹ cho phép xử lý và phân tích dữ liệu thời gian thực được lưu trữ trong Kafka.

- **Các đặc điểm chính:**
    - **Không có dependency bên ngoài:** Chỉ phụ thuộc vào Apache Kafka, tận dụng trực tiếp mô hình phân vùng (partitioning) của Kafka để mở rộng theo chiều ngang.
    - **Xử lý từng bản ghi (One-record-at-a-time):** Đạt độ trễ xử lý cực thấp ở mức miligiây.
    - **Quản lý trạng thái cục bộ (Fault-tolerant Local State):** Hỗ trợ lưu trữ trạng thái tại chỗ có khả năng chịu lỗi, phục vụ cho các thao tác phức tạp như join hay tổng hợp dữ liệu.
    - **Giao diện lập trình linh hoạt:** Cung cấp cả **Streams DSL** (cấp cao) và **Processor API** (cấp thấp).
    - **Cam kết Exactly-once:** Đảm bảo xử lý dữ liệu chính xác một lần trên toàn bộ đường ống.

---

## 2. Cấu trúc Topo xử lý luồng (Stream Processing Topology)

```
[ Source processor ]      [ Source processor ]
                 \                   /          \
            stream\            stream/           \stream
                   v                v             v
             [ Stream processor ]             [ Stream processor ]
                   \                              /
             stream \                            / stream
                     v                          v
                            [ Stream processor ]
                                     |
                              stream |
                                     v
                             [ Sink processor ]

                       === PROCESSOR TOPOLOGY ===
```

- **Stream (Luồng dữ liệu):** Khái niệm trừu tượng cốt lõi nhất, đại diện cho một tập dữ liệu không giới hạn và liên tục cập nhật. Được thể hiện dưới dạng một chuỗi bản ghi (key-value) bất biến, có thứ tự và chịu lỗi.

- **Processor Topology:** Đồ thị logic mô tả luồng xử lý, gồm các nút (**Stream Processors**) kết nối với nhau bằng các cạnh (**Streams**).

- **Stream Processor:** Nút xử lý dữ liệu trong topo, nhận từng bản ghi từ các nút phía trước, áp dụng thao tác biến đổi và phát ra các bản ghi cho các nút tiếp theo.

- **Hai nút đặc biệt:**
    - **Source Processor:** Nút đầu vào không có nút phía trên, đọc dữ liệu trực tiếp từ các topic Kafka.
    - **Sink Processor:** Nút đầu ra không có nút phía dưới, ghi dữ liệu kết quả trực tiếp vào topic Kafka.

---

## 3. Các khái niệm về Thời gian (Time Semantics)

- **Event time (Thời gian sự kiện):** Thời điểm sự kiện thực sự phát sinh tại nguồn (ví dụ: mốc thời gian thiết bị GPS ghi nhận vị trí).

- **Processing time (Thời gian xử lý):** Thời điểm ứng dụng tiêu thụ và xử lý bản ghi đó (có thể chậm hơn mốc thời gian sự kiện vài miligiây hoặc nhiều giờ).

- **Ingestion time (Thời gian nạp):** Mốc thời gian bản ghi được broker ghi nối tiếp vào topic partition.

- **Stream time:** Thời gian vận hành dựa trên dữ liệu, chỉ tiến lên khi có bản ghi mới đến thông qua `TimestampExtractor`.

- **Quy tắc kế thừa và tính toán Timestamp cho bản ghi đầu ra:**
    - _Thao tác phi trạng thái (Stateless):_ Bản ghi đầu ra kế thừa trực tiếp timestamp từ bản ghi đầu vào tương ứng.
    - _Thao tác Join:_ Kết quả lấy `max(left.ts, right.ts)` cho stream-stream hoặc table-table join; lấy timestamp của bản ghi stream cho stream-table join.
    - _Thao tác Aggregation:_ Lấy timestamp lớn nhất (`max timestamp`) từ tất cả bản ghi đầu vào đóng góp vào kết quả đó.

---

## 4. Tính hai mặt giữa Stream và Table (Stream-Table Duality)

![[Pasted image 20261101231012.png]]

- **Bản chất:** Stream và Table là hai cách nhìn khác nhau trên cùng một tập dữ liệu.
    - **Stream as Table:** Stream có thể xem là một **changelog** (nhật ký thay đổi) của Table. Việc chạy lại stream từ đầu đến cuối sẽ khôi phục chính xác trạng thái của Table.
    - **Table as Stream:** Table là một **snapshot** (ảnh chụp trạng thái) chứa giá trị mới nhất cho mỗi key tại một thời điểm. Duyệt qua các phần tử trong Table sẽ tạo thành một Stream.

![[Pasted image 20261101231137.png]]

- **Ứng dụng:** Kafka Streams mô hình hóa trực tiếp khái niệm này qua các giao diện `KStream`, `KTable`, và `GlobalKTable` phục vụ nhân bản state store và thực hiện các truy vấn trạng thái.

---

## 5. Thao tác Tổng hợp (Aggregations) & Cửa sổ dữ liệu (Windowing)

- **Aggregation:** Thao tác kết hợp nhiều bản ghi đầu vào thành một bản ghi duy nhất, kết quả đầu ra **luôn luôn là một KTable**.

- **Windowing:** Cơ chế gom nhóm các bản ghi có cùng key theo các khoảng thời gian (cửa sổ) để thực hiện tính toán có trạng thái.

- **Grace Period (Khoảng thời gian ân hạn):** Xác định khoảng thời gian hệ thống sẵn sàng chờ các bản ghi đến muộn (out-of-order). Bản ghi đến sau mốc `end_of_window + grace_period` sẽ bị loại bỏ.

---

## 6. Quản lý Trạng thái (State Stores) & Interactive Queries

- **State Stores:** Kho lưu trữ trạng thái cục bộ (dạng in-memory hashmap hoặc RocksDB persistent store) được tích hợp trong từng tác vụ xử lý để phục vụ thao tác có trạng thái.

- **Interactive Queries:** Tính năng cho phép các ứng dụng hoặc tiến trình bên ngoài thực hiện truy vấn trực tiếp (chỉ đọc) vào các State Stores nội bộ của ứng dụng.

---

## 7. Cam kết Xử lý & Xử lý dữ liệu không đúng thứ tự (Processing Guarantees & Out-of-Order)

- **Exactly-Once Semantics (EOS):** Đảm bảo xử lý chính xác một lần nhờ cơ chế giao dịch nguyên tử giữa việc commit offset đầu vào, cập nhật state store và ghi dữ liệu ra topic đầu ra. Cấu hình khuyến nghị là `EXACTLY_ONCE_V2` (từ Kafka 2.6.0 / broker 2.5.0 trở lên).

- **Out-of-Order Handling:** Xử lý dữ liệu đến sai thứ tự thời gian bằng việc kết hợp khoảng thời gian ân hạn trong Windowing hoặc sử dụng **Versioned State Stores** trong các thao tác Join.
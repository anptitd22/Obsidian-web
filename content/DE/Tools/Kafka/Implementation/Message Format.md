## 1. Tổng quan Kiến trúc (Overview)

- **Quy tắc cốt lõi:** Kafka luôn ghi và lưu trữ thông điệp theo lô (**Record Batch**).
    
- **Thành phần:** Một Record Batch chứa một hoặc nhiều Record (trường hợp tối thiểu suy biến là batch chỉ có 1 record).
    
- **Mã hóa số nguyên linh hoạt:** Sử dụng chuẩn mã hóa **`varint` / `varlong`** (tương tự Google Protobuf) cho các trường kích thước/delta trong Record để tối ưu dung lượng đĩa.
    

## 2. Định dạng Lô Bản Ghi (Record Batch Format)

```text
baseOffset: int64
batchLength: int32
partitionLeaderEpoch: int32
magic: int8 (current magic value is 2)
crc: uint32
attributes: int16
    bit 0~2:
        0: no compression
        1: gzip
        2: snappy
        3: lz4
        4: zstd
    bit 3: timestampType
    bit 4: isTransactional (0 means not transactional)
    bit 5: isControlBatch (0 means not a control batch)
    bit 6: hasDeleteHorizonMs (0 means baseTimestamp is not set as the delete horizon for compaction)
    bit 7~15: unused
lastOffsetDelta: int32
baseTimestamp: int64
maxTimestamp: int64
producerId: int64
producerEpoch: int16
baseSequence: int32
recordsCount: int32
records: [Record]
```

|**Trường (Field)**|**Kiểu dữ liệu**|**Ý nghĩa kỹ thuật**|
|---|---|---|
|`baseOffset`|`int64`|Offset của bản ghi đầu tiên trong batch.|
|`batchLength`|`int32`|Số byte từ sau trường này đến hết batch. Tổng size thực tế = `batchLength + 12 bytes` (8 bytes `baseOffset` + 4 bytes `batchLength`).|
|`partitionLeaderEpoch`|`int32`|Định danh thế hệ của broker leader (không tính vào CRC để tránh broker phải tính lại CRC mỗi khi gán giá trị).|
|`magic`|`int8`|Phiên bản định dạng message format (hiện tại là `2`, áp dụng từ Kafka 0.11+).|
|`crc`|`uint32`|Mã kiểm tra lỗi CRC-32C (Castagnoli), tính từ trường `attributes` đến hết batch. Phải parse `magic` trước để biết cách đọc các byte trước đó.|
|`attributes`|`int16`|Cờ cấu hình bit-mask:<br><br>  <br><br>• **Bit 0–2:** Thuật toán nén (`0`: None, `1`: GZIP, `2`: Snappy, `3`: LZ4, `4`: ZSTD).<br><br>  <br><br>• **Bit 3:** `timestampType`.<br><br>  <br><br>• **Bit 4:** `isTransactional` (có thuộc transaction không).<br><br>  <br><br>• **Bit 5:** `isControlBatch` (có phải control batch không).<br><br>  <br><br>• **Bit 6:** `hasDeleteHorizonMs` (dùng cho compaction).<br><br>  <br><br>• **Bit 7–15:** Dự phòng (unused).|
|`lastOffsetDelta`|`int32`|Độ chênh lệch offset của bản ghi cuối so với `baseOffset`. Offset bản ghi cuối = `baseOffset + lastOffsetDelta`.|
|`baseTimestamp`|`int64`|Timestamp của bản ghi đầu tiên trong batch.|
|`maxTimestamp`|`int64`|Timestamp lớn nhất trong batch.|
|`producerId`|`int64`|ID định danh duy nhất của Producer (phục vụ Idempotence / Transaction).|
|`producerEpoch`|`int16`|Epoch của Producer (chống zombie instances).|
|`baseSequence`|`int32`|Số thứ tự (sequence number) của bản ghi đầu tiên do Producer cấp.|
|`recordsCount`|`int32`|Số lượng records chứa trong batch.|
|`records`|`[Record]`|Mảng các bản ghi (nếu bật nén, toàn bộ phần dữ liệu records này được nén liền khối ngay sau `recordsCount`).|

## 3. Hành vi của Batch trong Log Compaction

- **Bảo toàn Sequence Numbers & Offset:** Giữ nguyên `baseOffset`, `lastOffsetDelta`, và `baseSequence` của batch gốc ngay cả khi các record bên trong đã bị dọn sạch.
    
    - **Mục đích:** Khôi phục trạng thái Producer khi reload log, kiểm tra trùng lặp (deduplication) và tránh lỗi `OutOfSequence`.
        
    - **Hiện tượng Batch Rỗng:** Có thể xuất hiện các batch không chứa record nào (`recordsCount = 0`) nhưng vẫn được giữ lại trên đĩa để neo giữ sequence number cuối cùng của producer.
        
- **Thay đổi `baseTimestamp`:**
    
    - Không được bảo toàn tuyệt đối; sẽ thay đổi nếu record đầu tiên bị xóa đi do compaction.
        
    - Nếu batch chứa tombstone (payload null) hoặc aborted transaction marker, `baseTimestamp` được cập nhật thành thời điểm hết hạn để xóa hẳn (delete horizon) kèm cờ bit 6 (`hasDeleteHorizonMs`).
        

## 4. Control Batches (Lô Điều Khiển)

- **Bản chất:** Batch đặc biệt chứa đúng một **Control Record**.
    
- **Phạm vi sử dụng:** Dùng nội bộ giữa Broker và Consumer; **không bao giờ trả về tầng ứng dụng**.
    
    - Consumer dùng để lọc/bỏ qua các message thuộc transaction bị hủy (aborted).
        
    - Giao thức KRaft dùng để truyền metadata đồng thuận.
        
- **Cấu trúc Key của Control Record:**

    - `version`: `int16` (hiện tại là `0`).
        
    - `type`: `int16`:
        
        - `Type 0 (ABORT)`: Đánh dấu giao dịch bị hủy.
            
        - `Type 1 (COMMIT)`: Đánh dấu giao dịch thành công.
            
        - `Type 2 ~ 6`: Dùng nội bộ cho giao thức đồng thuận **KRaft**.
            
- **Value:** Dữ liệu opaque tùy thuộc theo từng `type`.

```text
version: int16 (current version is 0)
type: int16 (the control record types are in the table below)
```

## 5. Định dạng Từng Bản Ghi (Record On-Disk Format)

```text
length: varint
attributes: int8
    bit 0~7: unused
timestampDelta: varlong
offsetDelta: varint
keyLength: varint
key: byte[]
valueLength: varint
value: byte[]
headersCount: varint
Headers => [Header]
```

Mỗi record bên trong batch được lưu trữ với các trường:

- **`length` (`varint`):** Tổng kích thước byte của record.
    
- **`attributes` (`int8`):** Bit 0–7 hiện chưa sử dụng (unused).
    
- **`timestampDelta` (`varlong`):** Độ chênh lệch thời gian so với `baseTimestamp` của batch.
    
- **`offsetDelta` (`varint`):** Độ chênh lệch offset so với `baseOffset` của batch.
    
- **`keyLength` (`varint`) & `key` (`byte[]`):** Kích thước và mảng byte khóa.
    
- **`valueLength` (`varint`) & `value` (`byte[]`):** Kích thước và mảng byte giá trị.
    
- **`headersCount` (`varint`):** Số lượng header đi kèm.
    
- **`Headers` (`[Header]`):** Danh sách các cặp key-value bổ trợ.
    

## 6. Cấu trúc Record Header

```text
headerKeyLength: varint
headerKey: String
headerValueLength: varint
Value: byte[]
```

- **Các trường:**
    
    - `headerKeyLength`: `varint`
        
    - `headerKey`: `String` (được đảm bảo luôn **khác null**)
        
    - `headerValueLength`: `varint`
        
    - `Value`: `byte[]` (có thể mang giá trị **null**)
        
- **Đặc tính:** Thứ tự xuất hiện của các headers được giữ nguyên vẹn từ lúc Produce cho đến khi Consume.
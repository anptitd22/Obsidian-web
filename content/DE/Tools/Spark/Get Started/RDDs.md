## 1. Khái niệm cốt lõi (Core Concepts)

- **Resilient Distributed Dataset (RDD)**:
    - Là trừu tượng chính trong Spark ở high level, đại diện cho tập hợp dữ liệu phân tán trên các node của cụm, cho phép xử lý song song.
    - Có khả năng **tự động khôi phục khi lỗi** (fault-tolerant).
    - Có thể được **lưu tạm (persist/cache)** trong bộ nhớ để tái sử dụng hiệu quả giữa các operator.

## 2. Khởi tạo & Tạo RDD (Initialization & Creation)

### A. SparkContext

- Mọi Spark application đều cần khởi tạo đối tượng [[SparkContext]] (thông qua `SparkConf`) để kết nối với cụm.

- Trong Spark Shell (`bin/pyspark` hoặc `bin/spark-shell`), đối tượng `sc` đã được khởi tạo sẵn.

### B. Hai cách tạo RDD

1. **Parallelized Collections (Tập hợp song song)**:
    - Tạo bằng cách gọi `sc.parallelize(collection)` trên một tập hợp có sẵn trong [[Driver Program]].
    - **Partitions (Phân đoạn)**: Xác định số lượng công việc chạy song song (thường chọn 2–4 partitions cho mỗi CPU).
2. **External Datasets (Tập dữ liệu ngoài)**:
    - Đọc từ tệp tin qua `sc.textFile("path")` (hỗ trợ đường dẫn local, `hdfs://`, `s3a://`, thư mục, tệp nén `.gz` hoặc wildcard).
    - `wholeTextFiles`: Đọc thư mục chứa nhiều tệp nhỏ, trả về cặp `(filename, content)`.
    - Các định dạng khác: `sequenceFile`, `hadoopRDD` / `newAPIHadoopRDD`, `objectFile`.

## 3. Thao tác trên RDD (RDD Operations)

### A. Transformations (Phép biến đổi)

- **Đặc tính**: **Lazy Evaluation (Tính toán lười biếng)** — Spark không tính toán kết quả ngay mà chỉ ghi lại chuỗi biến đổi. Việc tính toán chỉ thực sự diễn ra khi một **Action** được gọi.

- **Các Transformation phổ biến**:
    - `map(func)`: Áp dụng hàm lên từng phần tử.
    - `filter(func)`: Lọc các phần tử thỏa mãn điều kiện.
    - `flatMap(func)`: Áp dụng hàm trả về từ 0 đến nhiều phần tử.
    - `union(other)` / `intersection(other)` / `distinct()`: Phép hợp, giao và lấy phần tử duy nhất.
    - `groupByKey()`: Nhóm các cặp (K, V) theo khóa _(Lưu ý: Nếu dùng để tính tổng/gộp, dùng_ _reduceByKey_ _hoặc_ _aggregateByKey_ _sẽ cho hiệu năng tốt hơn nhiều)_.
    - `reduceByKey(func)`: Gộp giá trị theo khóa bằng hàm gộp.
    - `join(other)` / `cogroup(other)`: Kết nối các RDD dạng cặp (K, V).
    - `coalesce(numPartitions)` / `repartition(numPartitions)`: Giảm hoặc tái phân bổ số lượng partition.

### B. Actions (Hành động)

- **Đặc tính**: Chạy tính toán trên RDD và trả về kết quả cho [[Driver Program]] hoặc ghi dữ liệu ra hệ thống lưu trữ.

- **Các Action phổ biến**:
    - `reduce(func)`: Gộp tất cả phần tử bằng hàm giao hoán và kết hợp.
    - `collect()`: Trả về toàn bộ phần tử dưới dạng mảng về Driver _(Cảnh báo: Có thể tràn bộ nhớ Driver nếu RDD quá lớn)_.
    - `count()`: Trả về số lượng phần tử.
    - `first()` / `take(n)`: Lấy phần tử đầu tiên hoặc $n$ phần tử đầu.
    - `saveAsTextFile(path)` / `saveAsSequenceFile(path)` / `saveAsObjectFile(path)`: Lưu RDD ra tệp.
    - `countByKey()`: Đếm số lượng phần tử theo từng khóa.
    - `foreach(func)`: Chạy hàm trên từng phần tử (thường dùng cho side-effects như cập nhật Accumulator).

---

## 4. Quá trình Shuffle (Shuffle Operations)

- **Khái niệm**: Là cơ chế phân phối lại dữ liệu giữa các partition/node để gom nhóm dữ liệu theo cách mới (ví dụ khi gom nhóm theo key).

- Để tổ chức dữ liệu cho quá trình [[Shuffle]], Spark chia công việc thành hai tập hợp task:

	1. **Phía Map (Map tasks)**: Các task xử lý dữ liệu ban đầu và tổ chức lại các bản ghi theo partition đích. Kết quả được giữ trong bộ nhớ; khi bộ nhớ đầy, dữ liệu sẽ được sắp xếp theo partition đích và ghi ra một tệp tạm trên đĩa.
	2. **Phía Reduce (Reduce tasks)**: Các task ở giai đoạn sau sẽ đọc các khối dữ liệu đã được sắp xếp tương ứng từ tất cả các máy phía Map thông qua mạng để thực hiện tính toán/tổng hợp.

- **Chi phí hiệu năng**: Mất nhiều chi phí vì đòi hỏi I/O đĩa, ghi/đọc đĩa tạm, tràn bộ nhớ ,  tuần tự hóa (serialization) và truyền tải qua mạng.

- **Các phép toán gây Shuffle**: `repartition`, `coalesce`, các phép toán `'ByKey` (trừ đếm), `join`, `cogroup`.

## 5. Lưu trữ & Lưu tạm RDD (Persistence / Caching)

- **Công dụng**: Giữ lại các partition đã tính toán trong bộ nhớ/đĩa để tái sử dụng ở các Action sau, giúp tăng tốc độ đáng kể (thường hơn 10 lần).

- **Cơ chế khôi phục**: Nếu một partition bị mất, Spark sẽ tự động tính toán lại dựa trên chuỗi transformation ban đầu.

- **Các Storage Level phổ biến**:
    - `MEMORY_ONLY` _(mặc định)_: Lưu RDD dạng Java object trong JVM.
    - `MEMORY_AND_DISK`: Nếu vượt quá dung lượng bộ nhớ, tràn các partition thừa ra đĩa.
    - `MEMORY_ONLY_SER` / `MEMORY_AND_DISK_SER`: Lưu dạng đối tượng đã tuần tự hóa (tiết kiệm bộ nhớ nhưng tốn CPU hơn khi đọc).
    - `DISK_ONLY`: Chỉ lưu trên đĩa.
    - `MEMORY_ONLY, MEMORY_ONLY_2, MEMORY_AND_DISK, MEMORY_AND_DISK_2, DISK_ONLY, DISK_ONLY_2 và DISK_ONLY_3`.

- **Giải phóng bộ nhớ**: Spark tự giải phóng theo thuật toán LRU hoặc có thể gọi `unpersist()` thủ công.


## Components

Các Spark application chạy như các tập hợp tiến trình độc lập trên một cluster, được điều phối bởi SparkContext object trong program chính của bạn (được gọi là _driver program_).

Cụ thể, để chạy trên một cluster, SparkContext có thể kết nối với nhiều loại _cluster managers_ (hoặc standalone cluster manager của Spark, YARN hoặc Kubernetes), các cluster managers này sẽ phân bổ tài nguyên cho các applications. Sau khi kết nối, Spark sẽ lấy các _executors_ trên các nodes trong cluster, đó là các tiến trình chạy các computations và lưu trữ dữ liệu cho application của bạn. Tiếp theo, nó gửi application code của bạn (được định nghĩa bởi các tệp JAR hoặc Python được truyền cho SparkContext) đến các executors. Cuối cùng, SparkContext gửi các task đến các executors để chạy.

![[spark_components.png]]

>[!note]
>Dataproc thì cluster manager là YARN

 Mỗi application có các tiến trình executor riêng, hoạt động liên tục trong suốt quá trình chạy application và thực hiện các task trên nhiều luồng. 
 
 - Điều này mang lại lợi ích là cô lập các application với nhau, cả về phía scheduling (mỗi driver schedules cho các task của riêng nó) và phía executor (các task từ các ứng dụng khác nhau chạy trong các JVM khác nhau). 

 - Tuy nhiên, điều này cũng có nghĩa là dữ liệu không thể được chia sẻ giữa các Spark application khác nhau (các instances của SparkContext) mà không cần ghi vào hệ thống lưu trữ bên ngoài.

Spark không phụ thuộc vào cluster manager nào. Miễn là nó có thể có được các tiến trình executor và các tiến trình này có thể giao tiếp với nhau, thì việc chạy Spark tương đối dễ dàng ngay cả trên một cluster manager cũng hỗ trợ các ứng dụng khác (ví dụ: YARN/Kubernetes).

The driver program phải lắng nghe và chấp nhận các kết nối đến từ các executors của nó trong suốt vòng đời của nó (ví dụ: xem `spark.driver.port` trong phần cấu hình mạng [Spark Configuration](https://spark.apache.org/docs/3.5.9/configuration.html#spark-configuration). Do đó, he driver program phải có thể được truy cập qua mạng từ các worker nodes.

Vì driver schedules tasks trên cluster, nên nó cần được chạy gần các worker nodes, tốt nhất là trên cùng một mạng cục bộ (LAN). 

- Nếu bạn muốn gửi yêu cầu đến cluster từ xa, tốt hơn hết là nên mở một giao tiếp từ xa (RPC) đến driver và để nó submit operations từ gần đó hơn là chạy driver ở xa các worker nodes, có thể dùng [[Spark Connect]]

## Submitting Applications

Các application có thể được gửi đến cluster thuộc bất kỳ loại nào bằng cách sử dụng tập lệnh `spark-submit`. Hướng dẫn [[Submitting Applications]] cách thực hiện việc này.

## Monitoring

Mỗi driver program đều có web UI, thường là trên cổng 4040, hiển thị thông tin về các task đang chạy, executor và mức sử dụng bộ nhớ. Chỉ cần truy cập `http://<driver-node>:4040` trong trình duyệt web để truy cập UI này. Hướng dẫn [[Monitoring and Instrumentation]] cũng mô tả các tùy chọn giám sát khác.

## Job Scheduling

Spark cho phép kiểm soát việc phân bổ tài nguyên cả giữa các application (ở cấp độ cluster manager) và trong nội bộ application (nếu nhiều computations đang diễn ra trên cùng một SparkContext). Phần tổng quan về [[Job Scheduling]] mô tả điều này chi tiết hơn.

## Glossary

Bảng sau đây tóm tắt các thuật ngữ bạn sẽ thấy được sử dụng để chỉ các khái niệm nhóm:

| **Term**                                       | **Meaning**                                                                                                                                                                                                                                                                                                                                | **Application**                                                                                         |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| **Application** _(Ứng dụng)_                   | Chương trình do người dùng xây dựng dựa trên Spark. Bao gồm một chương trình điều khiển (driver) và các bộ thực thi (executors) trên cụm máy chủ (cluster).                                                                                                                                                                                | Chương trình ứng dụng Spark tổng thể do bạn viết.                                                       |
| **Application jar** _(File jar của ứng dụng)_  | Một file `.jar` chứa Spark application của người dùng. Trong một số trường hợp, người dùng sẽ muốn tạo một "uber jar" (hoặc fat jar) chứa cả ứng dụng lẫn các thư viện phụ thuộc (dependencies). Tuy nhiên, file jar này **không được chứa** các thư viện của Hadoop hay Spark, vì chúng sẽ được thêm vào lúc chương trình chạy (runtime). | Dùng để đóng gói mã nguồn và thư viện bổ sung khi nộp (submit) application.                             |
| **Driver program** _(Chương trình điều khiển)_ | Tiến trình (process) chạy hàm `main()` của ứng dụng và khởi tạo đối tượng `SparkContext`.                                                                                                                                                                                                                                                  | Nơi điều phối toàn bộ luồng xử lý và quản lý tiến trình chung của ứng dụng.                             |
| **Cluster manager** _(Trình quản lý cụm)_      | Một dịch vụ bên ngoài có nhiệm vụ cấp phát tài nguyên trên cụm máy chủ (ví dụ: Standalone manager, YARN, Kubernetes).                                                                                                                                                                                                                      | Hệ thống chịu trách nhiệm phân bổ CPU/RAM trên toàn cụm cho Spark.                                      |
| **Deploy mode** _(Chế độ triển khai)_          | Phân biệt nơi mà tiến trình Driver được khởi chạy. Ở chế độ **"cluster"**, hệ thống sẽ chạy Driver ngay bên trong cụm máy chủ. Ở chế độ **"client"**, máy gửi lệnh (submitter) sẽ chạy Driver bên ngoài cụm máy chủ.                                                                                                                       | Xác định vị trí vật lý mà tiến trình Driver đứng để điều phối công việc.                                |
| **Worker node** _(Nut công việc / Máy nút)_    | Bất kỳ máy tính (node) nào trong cụm có khả năng chạy mã nguồn của ứng dụng.                                                                                                                                                                                                                                                               | Máy tính vật lý hoặc máy ảo đóng vai trò xử lý dữ liệu trong cụm.                                       |
| **Executor** _(Bộ thực thi)_                   | Một tiến trình được khởi tạo cho một ứng dụng trên Worker node, chịu trách nhiệm chạy các nhiệm vụ (tasks) và lưu trữ dữ liệu trên bộ nhớ (RAM) hoặc ổ đĩa xuyên suốt các nhiệm vụ đó. Mỗi ứng dụng sẽ có các Executor riêng biệt.                                                                                                         | Tiến trình trực tiếp ăn tài nguyên CPU/RAM để tính toán và lưu dữ liệu tạm thời.                        |
| **Task** _(Nhiệm vụ)_                          | Một đơn vị công việc nhỏ nhất sẽ được gửi đến cho một Executor xử lý.                                                                                                                                                                                                                                                                      | Công việc tính toán xử lý trên một luồng (thread) duy nhất của Executor.                                |
| **Stage** _(Giai đoạn)_                        | Mỗi Job sẽ được chia thành các tập hợp nhiệm vụ nhỏ hơn gọi là Stage. Các Stage này phụ thuộc lẫn nhau (tương tự như giai đoạn Map và Reduce trong MapReduce). Bạn sẽ thấy thuật ngữ này trong log của Driver.                                                                                                                             | Các bước trung gian chia nhỏ từ Job, thường tách biệt nhau bởi các thao tác xáo trộn dữ liệu (Shuffle). |
| **Job** _(Đợt tính toán / Tác vụ)_             | Một phép tính toán song song bao gồm nhiều Task, được sinh ra để phản hồi lại một thao tác hành động (_Spark action_ - ví dụ: `save`, `collect`). Bạn sẽ thấy thuật ngữ này xuất hiện trong log của Driver.                                                                                                                                | Tập hợp toàn bộ quá trình tính toán khi bạn kích hoạt một action trong Spark.                           |

## 1. What is UDF ?

UDF là viết tắt của User-Defined Function (Hàm do người dùng định nghĩa).

UDF cho phép bạn định nghĩa logic tùy chỉnh mà Spark có thể áp dụng cho các cột của DataFrame.

```
UDF
│
├── User-defined function
│
├── Useful for custom logic
│
├── Don't use when Spark already has a native function
│
├── Python UDF
│     └── Custom Python logic
│
├── Pandas UDF
│     └── Vectorized batch processing
│
├── Arrow UDF
│     └── PyArrow-based processing
│
├── Main concerns
│     ├── Python execution
│     ├── Serialization
│     ├── Optimization boundaries
│     └── Dependency management
│
└── Production rule
      └── Prefer Spark-native expressions
```

Ví dụ:

```
name
----------------
Alice
Bob
Charlie
```

Chúng tôi muốn tính độ dài của mỗi tên thì Spark đã cung cấp sẵn:

```
length("name")
```

Nhưng hãy tưởng tượng rằng sự chuyển đổi đó là điều mà Spark không cung cấp sẵn.

Chúng ta có thể viết:

```python
def custom_logic(value):
    return ...
```

và chuyển đổi nó thành một hàm UDF của Spark.

Về mặt khái niệm:

```
DataFrame
   │
   ▼
Column
   │
   ▼
Python UDF
   │
   ▼
Custom Python Logic
   │
   ▼
Result Column
```

Hiện tại, PySpark hỗ trợ một số loại hàm UDF Python, bao gồm **scalar Python UDFs, Pandas UDFs, and Arrow UDFs**.

## 2. Why Do We Need UDFs

Các hàm tích hợp sẵn của Spark bao gồm hầu hết các transformations thông dụng:

```python
from pyspark.sql.functions import (
    col,
    lower,
    upper,
    trim,
    regexp_replace,
    when,
    length
)
```

Ví dụ:

```python
df.withColumn(
    "clean_name",
    lower(trim(col("name")))
)
```

 Nhưng đôi khi logic nghiệp vụ của bạn lại chuyên biệt hơn.

```
Customer risk classification
Custom address normalization
Domain-specific text processing
Specialized mathematical calculation
Legacy Python business logic
Custom feature engineering
```

Giả sử tổ chức của bạn có một hàm Python phức tạp:

```python
def calculate_customer_segment(age, income, transactions):
    ...
```

Nếu không có biểu thức Spark gốc phù hợp, UDF có thể là lựa chọn thích hợp.

> UDF là một lối thoát cho logic tùy chỉnh — chứ không phải là cách mặc định để transformSpark DataFrame.

## 3. Creating Your First PySpark UDF

Tạo 1 UDF đơn giản:

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import IntegerType
@udf(returnType=IntegerType())
def name_length(name):
    if name is None:
        return None
    return len(name)
```

Bây giờ hãy tạo một DataFrame:

```python
data = [
    ("Alice",),
    ("Bob",),
    ("Charlie",),
    (None,)
]
df = spark.createDataFrame(data, ["name"])
```

Áp dụng hàm do UDF:

```python
result = df.withColumn(
    "name_length",
    name_length("name")
)
result.show()
```

Kết quả:

```console
+-------+-----------+
|   name|name_length|
+-------+-----------+
|  Alice|          5|
|    Bob|          3|
|Charlie|          7|
|   NULL|       NULL|
+-------+-----------+
```

UDF hoạt động giống như một biểu thức cột trong Spark.

## 4. UDF Using a Normal Python Function

Bạn không nhất thiết phải sử dụng cú pháp decorator.

```python
def name_length(name):
    if name is None:
        return None
  return len(name)
name_length_udf = udf(
    name_length,
    IntegerType()
)
```

Sau đó:

```python
df.withColumn(
    "name_length",
    name_length_udf("name")
)
```

Cả hai phương pháp đều tạo ra một UDF cho Spark.

`returnType` rất quan trọng vì Spark cần biết kiểu dữ liệu SQL trả về. API hiện tại chấp nhận hoặc là Spark `DataType` hoặc là chuỗi kiểu được định dạng theo DDL.

## 5. Using Multiple Columns

Một UDF có thể chấp nhận nhiều cột.

Giả sử:

```
age    income
30     100000
40     150000
```

Tạo:

```python
@udf(returnType="double")
def risk_score(age, income):
    if age is None or income is None:
        return None
  return income / age
```

Sử dụng:

```python
df.withColumn(
    "risk_score",
    risk_score(
        col("age"),
        col("income")
    )
)
```

Về mặt khái niệm:

```
age ───────┐
           │
income ────┼──► Python UDF ──► risk_score
```

## 6. UDF Return Types

Spark cần biết giá trị mà hàm UDF trả về.

Ví dụ:

### Integer

```python
@udf(returnType="int")
def calculate_score(x):
    return x * 10
```

### String

```python
@udf(returnType="string")
def normalize_name(x):
    return x.lower()
```

### Double

```python
@udf(returnType="double")
def calculate_ratio(a, b):
    return a / b
```

### Boolean

```python
@udf(returnType="boolean")
def is_valid(age):
    return age >= 18
```

Bạn cũng có thể sử dụng các Spark type objects:

```python
from pyspark.sql.types import IntegerType
@udf(returnType=IntegerType())
def square(x):
    return x * x
```

## 7. UDFs With Complex Return Types

Các UDF không bị giới hạn ở các giá trị nguyên thủy.

Ví dụ:

```python
from pyspark.sql.types import ArrayType, StringType
@udf(returnType=ArrayType(StringType()))
def split_words(text):
    if text is None:
        return None
    return text.split()
```

Sau đó:

```python
df.withColumn(
    "words",
    split_words("description")
)
```

Kết quả:

```
description             words
------------------------------------------------
"Spark is fast"         ["Spark", "is", "fast"]
```

Tuy nhiên, trước khi tạo một UDF phức tạp, hãy kiểm tra xem Spark đã có sẵn hàm gốc nào cho thao tác đó hay chưa.

## 8. The Most Important Rule: Prefer Built-in Functions

Đây có lẽ là quy tắc quan trọng nhất trong toàn bộ bài viết này:

>Nếu Spark đã có sẵn hàm tích hợp giải quyết vấn đề, hãy ưu tiên sử dụng hàm tích hợp đó thay vì UDF của Python.

Giả sử bạn muốn chuyển đổi tên thành chữ thường.

```python
@udf("string")
def lower_name(name):
    return name.lower()
```

Nhưng Spark đã cung cấp sẵn:

```python
from pyspark.sql.functions import lower
df.withColumn(
    "lower_name",
    lower("name")
)
```

Phiên bản thứ hai thường được ưa chuộng hơn.

Vì Spark hiểu được các cách diễn đạt gốc.

Nó có thể suy luận về chúng như một phần của kế hoạch truy vấn và áp dụng các tối ưu hóa SQL/DataFrame thông thường của mình.

## 9. UDF vs Built-in Function

Ta có:

```python
df.withColumn(
    "clean_name",
    lower(trim(col("name")))
)
```

So với:

```python
@udf("string")
def clean_name(name):
    return name.strip().lower()
df.withColumn(
    "clean_name",
    clean_name("name")
)
```

Cả 2 đều cho kết quả như nhau nhưng chúng không tương đương nhau từ góc độ thực thi của Spark.

### Built-in function

```
DataFrame
   │
   ▼
Spark Expression
   │
   ▼
Catalyst Optimizer
   │
   ▼
Optimized Physical Plan
   │
   ▼
Execution
```

### Python UDF

```
DataFrame
   │
   ▼
Python UDF
   │
   ▼
Python Worker
   │
   ▼
Python Logic
   │
   ▼
Spark Execution
```

Đường dẫn thứ hai đưa ra thêm một ranh giới thực thi.

## 10. Why Python UDFs Can Be Slow

Trong quá khứ, một trong những vấn đề hiệu năng lớn nhất của các UDF trong Python là việc trao đổi dữ liệu giữa công cụ Spark dựa trên JVM và các tiến trình Python.

Về mặt khái niệm:

```
              Spark JVM
                 │
                 │ serialize
                 ▼
          Python Worker
                 │
                 │ execute Python
                 ▼
          Python Function
                 │
                 │ return result
                 ▼
             Spark JVM
```

Ranh giới đó có thể phát sinh chi phí phát sinh.

Điều này đặc biệt quan trọng khi:

```
Dataset = billions of rows
        +
UDF = expensive Python logic
```

Ngay cả một khoản chi phí nhỏ phát sinh trên mỗi hàng cũng có thể trở nên đáng kể.

## 11. An Important Update: Arrow-Optimized Python UDFs

Phiên bản PySpark hiện đại đã cải thiện được khía cạnh này.

Trong Spark 4.2, các hàm UDF Python thông thường sử dụng Arrow để **serialization/deserialization** theo mặc định. Tuy nhiên, điều này không nên nhầm lẫn với hàm UDF Pandas: một hàm UDF Python thông thường được tối ưu hóa bằng Arrow vẫn xử lý các giá trị từng hàng một bằng cách sử dụng các Python object thông thường.

Về mặt khái niệm:

```
Traditional exchange
	Spark JVM
	   │
	   ▼
	Serialization
	   │
	   ▼
	Python
	   │
	   ▼
	Serialization
	   │
	   ▼
	Spark JVM
```

Với tính năng tối ưu hóa Arrow:

```
Spark JVM
   │
   ▼
Apache Arrow
   │
   ▼
Python Worker
   │
   ▼
Python Logic
```

Arrow có thể giúp việc truyền dữ liệu hiệu quả hơn.

>Việc tối ưu hóa bằng Arrow không tự động biến một hàm UDF của Python thành một biểu thức Spark gốc.

Hàm Python của bạn vẫn là logic Python.

## 12. Pandas UDFs

Một lựa chọn khác là sử dụng **Pandas UDF**.

Các hàm Pandas UDF là các hàm UDF được vector hóa, sử dụng Apache Arrow để truyền dữ liệu và hoạt động trên các batch dữ liệu thay vì xử lý từng giá trị Python riêng lẻ tại một thời điểm.

Ví dụ:

```python
import pandas as pd
from pyspark.sql.functions import pandas_udf
@pandas_udf("string")
def uppercase_names(names: pd.Series) -> pd.Series:
    return names.str.upper()
```

Sử dụng:

```python
df.withColumn(
    "upper_name",
    uppercase_names("name")
)
```

Về mặt khái niệm:

```
Scalar Python UDF
row → Python → result
row → Python → result
row → Python → result

Pandas UDF
batch → Python/Pandas → batch result
```

Điều này có thể giảm thiểu chi phí gọi hàm Python đối với các khối lượng công việc được hưởng lợi từ xử lý vector hóa.

## 13. Scalar UDF vs Pandas UDF

...

## 14. Arrow UDFs

Phiên bản PySpark hiện đại cũng cung cấp các hàm UDF Arrow.

Thay vì nhận các giá trị Python thông thường hoặc Pandas Series, các  Arrow UDF có thể làm việc trực tiếp với các đối tượng pyarrow.Array.

Ví dụ:

```python
import pyarrow as pa
from pyspark.sql.functions import arrow_udf
@arrow_udf("string")
def to_upper(values: pa.Array) -> pa.Array:
    return pa.compute.ascii_upper(values)
```

Đây là một cách khác mà PySpark có thể tích hợp logic Python với biểu diễn dạng cột của Arrow.

Vậy nên, bức tranh tổng quan về UDF hiện đại trông như sau:

```
Python UDFs
│
├── Scalar Python UDF
│
├── Pandas UDF
│
└── Arrow UDF
```

## 15. UDFs and Catalyst Optimizer

Đây là điểm khiến nhiều cuộc phỏng vấn về PySpark trở nên thú vị.

Giả sử bạn viết

```python
df.withColumn(
    "x",
    custom_python_function("value")
)
```

Spark biết rằng một UDF tồn tại.

Nhưng nhìn chung, nó không thể kiểm tra hàm Python tùy ý của bạn và suy luận về logic bên trong của nó theo cùng một cách mà nó hiểu biểu thức Spark gốc.

So:

```python
upper(col("name"))
```

Với:

```python
my_python_udf(col("name"))
```

Đầu tiên là một biểu thức Spark.

Phần thứ hai chứa logic Python tùy ý.

Điều này tạo ra một ranh giới tối ưu hóa.

## 16. Example: Why Native Expressions Are Better

Giả sử chúng ta cần:

```
price > 100
```

Native Spark:

```python
df.filter(col("price") > 100)
```

Spark hiểu được mệnh đề.

Bây giờ hãy tưởng tượng:

```python
@udf("boolean")
def expensive_filter(price):
    return price > 100
```

Sau đó:

```python
df.filter(expensive_filter("price"))
```

Spark không thể xử lý hàm Python đơn giản như biểu thức gốc.

Điều này có thể ảnh hưởng đến các cơ hội tối ưu hóa như predicate pushdown.

## 17. Predicate Pushdown and UDFs

Giả sử nguồn dữ liệu chứa:

```
1 billion records
```

và bạn cần:

```
price > 100
```

Với biểu thức gốc:

```python
df.filter(col("price") > 100)
```

Tùy thuộc vào nguồn dữ liệu và truy vấn, Spark và nguồn dữ liệu cơ bản có thể đẩy bộ lọc đến gần nguồn dữ liệu hơn.

Về mặt khái niệm:

```
Storage
   │
   │ filter pushed closer
   ▼
Only relevant data
   │
   ▼
Spark
```

Với Python UDF:

```python
df.filter(my_filter_udf("price"))
```

Spark có khả năng phân tích ngữ nghĩa của hàm Python kém hơn.

Vì vậy, bạn có thể bỏ lỡ các cơ hội tối ưu hóa.

## 18. UDFs and Conditional Expressions

Còn một hành vi quan trọng khác cần hiểu rõ.

Các UDFs không cung cấp các đảm bảo rút gọn mạch kiểu SQL thông thường cho các biểu thức điều kiện/boolean.

Ví dụ, đừng cho rằng việc viết:

```python
df.filter(
    (col("value") != 0) &
    expensive_udf("value")
)
```

Điều này có nghĩa là Spark sẽ luôn đánh giá giá trị != 0 trước và chỉ gọi hàm UDF đối với các hàng an toàn.

Tài liệu của PySpark ghi chú rõ ràng rằng các UDF không hỗ trợ các biểu thức điều kiện hoặc rút gọn vòng lặp trong các biểu thức boolean theo cách bạn mong đợi.

Nếu UDF của bạn có thể gặp lỗi khi nhận đầu vào đặc biệt, hãy đưa điều kiện an toàn vào chính hàm UDF đó.

Ví dụ:

```python
@udf("double")
def safe_divide(x):
    if x is None or x == 0:
        return None
    return 100 / x
```

## 19. Handling NULL Values

Hãy luôn suy nghĩ một cách rõ ràng về NULL.

Thực thi:

```python
@udf("string")
def normalize_name(name):
    return name.strip().lower()
```

Nếu:

```
name = NULL
```

Sau đó:

```
name.strip()
```

Lỗi

Tối ưu hơn:

```python
@udf("string")
def normalize_name(name):
    if name is None:
        return None
 return name.strip().lower()
```

Điều này đặc biệt quan trọng trong các quy trình prod vì giá trị NULL rất phổ biến trong các tập dữ liệu thực tế.

## 20. Deterministic vs Nondeterministic UDFs

Spark mặc định coi các UDF là có tính xác định.

Điều đó có nghĩa là Spark có thể giả định:

```
same input → same output
```

Giả sử:

```python
@udf("double")
def calculate(x):
    return x * 2
```

Đây là điều tất yếu.

Nhưng thử tưởng tượng:

```python
import random
@udf("double")
def random_value():
    return random.random()
```

Kết quả không mang tính xác định.

Bạn nên đánh dấu hàm UDF như vậy là không xác định:

```python
random_udf = udf(
    lambda: random.random(),
    "double"
).asNondeterministic()
```

Điều này rất quan trọng vì nếu không, trình tối ưu hóa của Spark có thể đưa ra các giả định về việc đánh giá lặp lại. Tài liệu PySpark hiện tại cung cấp rõ ràng hàm `asNondeterministic()` cho mục đích này.

## 21. Registering a UDF for SQL

Bạn có thể đăng ký một hàm UDF Python để có thể gọi nó từ Spark SQL.

```python
@udf("string")
def uppercase_name(name):
    if name is None:
        return None
    return name.upper()
```

Đăng ký:

```python
spark.udf.register(
    "uppercase_name",
    uppercase_name
)
```

Giờ đây SQL có thể gọi nó:

```sql
SELECT
    name,
    uppercase_name(name)
FROM people
```

Phương thức `UDFRegistration.register()` của PySpark hỗ trợ đăng ký các hàm Python và UDF để sử dụng như các hàm SQL.

## 22. A Real-World Example

Hãy tưởng tượng một tập dữ liệu về bảo hiểm:

```
customer_id
age
annual_income
claim_count
```

Chúng tôi muốn tính toán một điểm rủi ro tùy chỉnh.

```python
@udf("double")
def risk_score(age, income, claims):
  if age is None or income is None or claims is None:
        return None
    score = 0
  if age > 60:
      score += 20
  if income < 50000:
      score += 20
  if claims > 3:
      score += 40
  return float(score)
```

Áp dụng:

```python
result = df.withColumn(
    "risk_score",
    risk_score(
        col("age"),
        col("annual_income"),
        col("claim_count")
    )
)
```

Kết quả:

```
customer_id | age | income | claims | risk_score
-------------------------------------------------
101         | 65  | 40000  | 4      | 80
102         | 35  | 90000  | 1      | 0
```

Đây là một trường hợp sử dụng UDF hợp lý nếu logic thực sự không thể được diễn đạt một cách rõ ràng bằng các hàm tích hợp sẵn của Spark.

## 23. But Could We Avoid the UDF?

Thường thì đúng vậy.

Cùng một logic đó cũng có thể được diễn đạt bằng cách sử dụng:

```python
from pyspark.sql.functions import when, lit
result = df.withColumn(
    "risk_score",
    when(col("age") > 60, 20).otherwise(0)
    + when(col("annual_income") < 50000, 20).otherwise(0)
    + when(col("claim_count") > 3, 40).otherwise(0)
)
```

Điều đó thường là lựa chọn tốt hơn.

## 24. A Practical Decision Tree

Khi muốn viết một UDF, hãy tự hỏi những câu hỏi sau:

```
              Need custom logic?
                       │
                       ▼
          Does Spark have a built-in
               function for it?
                 /          \
               YES          NO
                │             │
                ▼             ▼
          Use Spark       Can the logic
          function        be expressed using
                          Spark SQL expressions?
                            /       \
                          YES       NO
                           │         │
                           ▼         ▼
                      Use native   Need Python?
                      expression      │
                                      ▼
                              Consider UDF
                                      │
                         ┌────────────┼────────────┐
                         ▼            ▼            ▼
                      Scalar       Pandas        Arrow
                       UDF          UDF           UDF
```

Mục tiêu không phải là:

> _"Never use UDFs."_

Mục tiêu là:

> **_Sử dụng the highest-level Spark-native abstraction để giải quyết vấn đề một cách chính xác._**

## 25. UDF Performance Hierarchy

Một mô hình tư duy thực tiễn hữu ích là:

```
Usually easiest for Spark to optimize
                │
                ▼
      Built-in Spark functions
                │
                ▼
       Spark SQL expressions
                │
                ▼
      Native Spark operations
                │
                ▼
       Pandas / Arrow UDFs
                │
                ▼
       Python UDFs
                │
                ▼
   Expensive external Python logic
```

Đây không phải là bảng xếp hạng chuẩn mực toàn cầu.

Hiệu năng thực tế phụ thuộc vào khối lượng công việc, cách triển khai, kiểu dữ liệu, hành vi xử lý theo lô và phiên bản Spark.

Nhưng xét về nguyên tắc kỹ thuật:

>Đừng chuyển sang học Python trừ khi bạn có lý do chính đáng.

## 26. UDFs and Serialization

Một Spark application phân tán có nhiều tiến trình.

Về mặt khái niệm:

```
Driver
  │
  ▼
Executors
  │
  ├── Task
  │
  ├── Task
  │
  └── Task
        │
        ▼
   Python Worker
```

Khi sử dụng UDF trong Python, Spark phải điều phối quá trình thực thi giữa môi trường chạy Spark dựa trên JVM và các tiến trình Python.

Việc tối ưu hóa Arrow hiện đại có thể cải thiện quá trình serialization/deserialization, nhưng ranh giới thực thi của Python vẫn rất quan trọng.

Đối với triển khai cụm, tất cả các thư viện Python cần thiết cũng phải có sẵn trên các máy thực thi. Điều này đặc biệt quan trọng đối với các hàm UDF Pandas/Arrow vì chúng phụ thuộc vào các gói như Pandas và PyArrow.

## 27. Common Mistake: Using UDF for Simple String Logic

Tệ:

```python
@udf("string")
def upper_name(name):
    return name.upper()
```

Tốt hơn:

```python
from pyspark.sql.functions import upper
df.withColumn(
    "upper_name",
    upper("name")
)
```

Spark đã biết cách thực hiện thao tác này.

## 28. Common Mistake: Using UDF for Date Operations

Tệ:

```python
@udf("int")
def get_year(date):
    return date.year
```

Thay vì:

```python
from pyspark.sql.functions import year
df.withColumn(
    "year",
    year("date")
)
```

Hãy sử dụng các hàm ngày/giờ của Spark bất cứ khi nào có thể.

## 29. Common Mistake: Calling External APIs Inside UDFs

Đây là một trong những mô hình prod phản tác dụng lớn nhất.

Tránh:

```python
@udf("string")
def call_api(customer_id):
    response = requests.get(...)
    return response.text
```

Why ?

Tưởng tượng:

```
100 million rows
       ×
100 million API calls
```

Bạn đã biến một job xử lý dữ liệu phân tán đến một máy khách API phân tán.

Các vấn đề có thể bao gồm:

- API rate limits
- retries
- network latency
- unpredictable execution time
- duplicate requests
- failed tasks
- external system overload
- difficult observability

Nếu cần bổ sung dữ liệu từ bên ngoài, hãy thiết kế nó như một quy trình làm việc riêng biệt có khả năng mở rộng thay vì đặt một yêu cầu API một cách mù quáng bên trong một UDF ở cấp hàng.

## 30. Common Mistake: Heavy Machine Learning Inside a Row UDF

Tránh tải lại mô hình nhiều lần:

```python
@udf("double")
def predict(features):
    model = load_model()
    return model.predict(features)
```

Mô hình này có thể được tải lặp đi lặp lại trên nhiều tiến trình/tác vụ Python khác nhau.

Đối với suy luận ML phân tán, hãy xem xét các kiến ​​trúc được thiết kế cho suy luận theo lô, các hàm UDF được vector hóa, phân phối mô hình hoặc các khả năng ML/suy luận của Spark tùy thuộc vào trường hợp sử dụng.

## 31. Common Mistake: Ignoring Python Dependencies

Giả sử:

```python
@pandas_udf("double")
def calculate(x):
    import some_library
    ...
```

Máy tính xách tay của bạn có thư viện đó. 

Còn máy thực thi của bạn thì có thể không. 

Do đó, bạn có thể thấy:

```
ModuleNotFoundError
```

Môi trường Spark prod cần có các thư viện Python nhất quán trên toàn bộ môi trường worker.

## 32. How to Debug a Slow UDF

Giả sử việc này mất:

```
2 minutes locally
```

nhưng:

```
2 hours on the cluster
```

Không nên tăng kích thước cụm ngay lập tức.

Khảo sát:

### Step 1 — Check the execution plan

```
df.explain("formatted")
```

### Step 2 — Look for Python execution

Kiểm tra physical plan của các node thực thi liên quan đến Python UDF.

### Step 3 — Check Spark UI

Look at:

```
Jobs  ↓Stages  ↓Tasks
```

Check:

- task duration
- input size
- shuffle
- spill
- executor behavior
- skew
- Python execution time where available

### Step 4 — Compare against a native implementation

Replace:

```
my_udf(...)
```

with an equivalent Spark expression if possible.

### Step 5 — Profile the UDF

PySpark provides profiling support for Python, Pandas, and Arrow UDFs.

### 33. UDF Profiling

For example, Spark provides a UDF profiler configuration:

```
spark.conf.set(
    "spark.sql.pyspark.udf.profiler",
    "perf"
)
```

Then execute the UDF workload.

Điều này có thể giúp xác định thời gian thực thi của Python UDF được dành cho những phần nào.

Để khắc phục sự cố trong môi trường prod, hãy kết hợp phân tích hiệu năng với:

```
Spark UI
+
Execution Plan
+
Application Logs
+
UDF Profiling
```

thay vì chỉ dựa vào một phép đo duy nhất.

Tài liệu PySpark hiện tại mô tả đây là ba loại chính của Python UDF.

## 36. Production Checklist

Trước khi triển khai một hàm do người dùng định nghĩa (UDF), hãy tự hỏi:

### Correctness

- Does it handle NULL?
- Is the return type correct?
- Is the function deterministic?
- Can it fail for special inputs?

### Performance

- Is there a built-in Spark function?
- Can I express the logic using Spark SQL?
- Does the UDF process huge volumes?
- Would vectorization help?
- Have I measured the actual performance?

### Distributed execution

- Are dependencies installed on executors?
- Does the function perform network calls?
- Does it load large models/files?
- Does it create unnecessary serialization overhead?

### Observability

- Did I inspect `explain()`?
- Did I inspect Spark UI?
- Did I profile the UDF if necessary?

## 37. The Production Engineer's Rule

Khi bạn nhìn thấy điều này:

```python
@udf(...)
def something(...):
    ...
```

Đừng hỏi:

> _"Is this UDF correct?"_

Hãy hỏi:

> **_"Why is this a UDF?"_**

Có lẽ có thể viết logic như sau:

```
when()
regexp_replace()
split()
transform()
filter()
aggregate()
expr()
```

or another native Spark expression.

If so, prefer that.

If not, then a UDF may be exactly the right tool.

### 38. The Mental Model

Hãy nhớ:

```
                Spark DataFrame
                       │
                       ▼
             Can Spark understand it?
                    /       \
                  YES        NO
                   │          │
                   ▼          ▼
             Native Spark    Python
               expression     UDF
                   │          │
                   ▼          ▼
              Optimizer     Python
              friendly      boundary
                   │          │
                   └────┬─────┘
                        ▼
                    Execution
```

## 39. Common Interview Questions

### 1. What is a UDF in PySpark?

A UDF is a user-defined function that allows custom Python logic to be applied to Spark DataFrame columns when built-in Spark expressions aren't sufficient.

### 2. Why are Python UDFs often slower than built-in functions?

Because Python UDFs introduce Python execution and a JVM-to-Python execution boundary. Built-in Spark expressions are understood directly by Spark's query engine and provide more optimization opportunities.

### 3. Should you always avoid UDFs?

No.

Use them when the required logic cannot reasonably be expressed using Spark's built-in functions or SQL expressions.

### 4. What is a Pandas UDF?

A Pandas UDF is a vectorized UDF that processes batches using Pandas and Apache Arrow rather than invoking Python logic independently for every individual value.

### 5. What is Arrow's role in PySpark UDFs?

Apache Arrow provides an efficient columnar representation for transferring data between Spark's JVM runtime and Python. In Spark 4.2, Arrow is also used by default for serialization/deserialization of regular Python UDFs.

### 6. Can a UDF accept multiple columns?

Yes.

```python
my_udf(
    col("age"),
    col("income")
)
```

### 7. Can UDFs return complex types?

Yes. UDFs can return supported Spark SQL types including arrays and structs.

### 8. What happens if a UDF is nondeterministic?

UDFs are deterministic by default. If the function is nondeterministic, such as one using random values, it should be marked using:

```python
.asNondeterministic()
```

### 9. Why should you avoid API calls inside UDFs?

Because a row-level distributed UDF can generate a huge number of external requests, leading to rate limiting, network latency, retries, failures, and unpredictable execution time.

### 10. How do you decide whether to use a UDF?

Use this order:

```python
Built-in Spark function
        ↓
Spark SQL expression
        ↓
Native DataFrame transformation
        ↓
Pandas/Arrow UDF
        ↓
Python UDF
```

Hãy chọn phương án đơn giản nhất giải quyết đúng vấn đề.
---
title : "Thiết kế bảng và tải dữ liệu"
date :  "`r Sys.Date()`" 
weight : 2 
chapter : false
pre : " <b> 2. </b> "
---
### **2. Thiết kế bảng và tải dữ liệu**

### **Nội dung:**

- Trước khi bắt đầu
- Tạo bảng
- Tải dữ liệu
- Khắc phục sự cố tải dữ liệu
- Bảo trì bảng tự động - ANALYZE và VACUUM
- Trước khi rời bỏ

### **2.1 Trước khi bắt đầu**

Bài lab này giả định rằng bạn đã khởi chạy một endpoint Amazon Redshift Serverless. Nếu bạn chưa làm như vậy, hãy xem phần [Bắt đầu](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab1) và làm theo hướng dẫn ở đó. Chúng ta sẽ sử dụng Amazon Redshift [QueryEditorV2](https://ap-southeast-2.console.aws.amazon.com/sqlworkbench/home?region=ap-southeast-2#/account/configuration) cho bài lab này.

### **2.2. Tạo bảng**

Amazon Redshift là một kho dữ liệu tuân thủ chuẩn ANSI SQL. Bạn có thể tạo bảng bằng các câu lệnh [CREATE TABLE](https://docs.aws.amazon.com/redshift/latest/dg/r_CREATE_TABLE_NEW.html) quen thuộc.

Để tạo 8 bảng từ mô hình dữ liệu TPC Benchmark, hãy sao chép các câu lệnh tạo bảng dưới đây và chạy chúng. Dưới đây là mô hình dữ liệu cho các bảng.

![image.png](/images/2/2-1.png)

```jsx
DROP TABLE IF EXISTS partsupp;
DROP TABLE IF EXISTS lineitem;
DROP TABLE IF EXISTS supplier;
DROP TABLE IF EXISTS part;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS customer;
DROP TABLE IF EXISTS nation;
DROP TABLE IF EXISTS region;

CREATE TABLE region (
  R_REGIONKEY bigint NOT NULL,
  R_NAME varchar(25),
  R_COMMENT varchar(152))
diststyle all;

CREATE TABLE nation (
  N_NATIONKEY bigint NOT NULL,
  N_NAME varchar(25),
  N_REGIONKEY bigint,
  N_COMMENT varchar(152))
diststyle all;

create table customer (
  C_CUSTKEY bigint NOT NULL,
  C_NAME varchar(25),
  C_ADDRESS varchar(40),
  C_NATIONKEY bigint,
  C_PHONE varchar(15),
  C_ACCTBAL decimal(18,4),
  C_MKTSEGMENT varchar(10),
  C_COMMENT varchar(117))
diststyle all;

create table orders (
  O_ORDERKEY bigint NOT NULL,
  O_CUSTKEY bigint,
  O_ORDERSTATUS varchar(1),
  O_TOTALPRICE decimal(18,4),
  O_ORDERDATE Date,
  O_ORDERPRIORITY varchar(15),
  O_CLERK varchar(15),
  O_SHIPPRIORITY Integer,
  O_COMMENT varchar(79))
distkey (O_ORDERKEY)
sortkey (O_ORDERDATE);

create table part (
  P_PARTKEY bigint NOT NULL,
  P_NAME varchar(55),
  P_MFGR  varchar(25),
  P_BRAND varchar(10),
  P_TYPE varchar(25),
  P_SIZE integer,
  P_CONTAINER varchar(10),
  P_RETAILPRICE decimal(18,4),
  P_COMMENT varchar(23))
diststyle all;

create table supplier (
  S_SUPPKEY bigint NOT NULL,
  S_NAME varchar(25),
  S_ADDRESS varchar(40),
  S_NATIONKEY bigint,
  S_PHONE varchar(15),
  S_ACCTBAL decimal(18,4),
  S_COMMENT varchar(101))
diststyle all;

create table lineitem (
  L_ORDERKEY bigint NOT NULL,
  L_PARTKEY bigint,
  L_SUPPKEY bigint,
  L_LINENUMBER integer NOT NULL,
  L_QUANTITY decimal(18,4),
  L_EXTENDEDPRICE decimal(18,4),
  L_DISCOUNT decimal(18,4),
  L_TAX decimal(18,4),
  L_RETURNFLAG varchar(1),
  L_LINESTATUS varchar(1),
  L_SHIPDATE date,
  L_COMMITDATE date,
  L_RECEIPTDATE date,
  L_SHIPINSTRUCT varchar(25),
  L_SHIPMODE varchar(10),
  L_COMMENT varchar(44))
distkey (L_ORDERKEY)
sortkey (L_RECEIPTDATE);

create table partsupp (
  PS_PARTKEY bigint NOT NULL,
  PS_SUPPKEY bigint NOT NULL,
  PS_AVAILQTY integer,
  PS_SUPPLYCOST decimal(18,4),
  PS_COMMENT varchar(199))
diststyle even;
```

![image.png](/images/2/2-2.png)

![image.png](/images/2/2-33.png)

### **2.3 Tải dữ liệu**

Trong bài lab này, bạn sẽ học các phương pháp sau để tải dữ liệu vào Redshift:

1. Tải dữ liệu từ S3 vào Redshift. Bạn sẽ sử dụng quy trình này để tải 7 bảng trong số 8 bảng đã tạo ở bước trên.
2. Tải dữ liệu từ tệp trên máy tính của bạn vào Redshift. Bạn sẽ sử dụng quy trình này để tải 1 bảng trong số 8 bảng đã tạo ở bước trên, cụ thể là bảng nation.

Chúng tôi chia nhỏ quy trình tải dữ liệu để giúp bạn làm quen với cả hai phương pháp.

- **Tải dữ liệu từ S3**

Lệnh [COPY](https://docs.aws.amazon.com/redshift/latest/dg/r_COPY.html) tải số lượng lớn dữ liệu hiệu quả từ Amazon S3 vào Amazon Redshift. Một lệnh **COPY** có thể tải dữ liệu từ nhiều tệp vào một bảng. Nó tự động tải dữ liệu song song từ tất cả các tệp có sẵn tại vị trí S3 bạn cung cấp.

Nếu bảng đích còn trống, lệnh **COPY** có thể thực hiện mã hóa nén cho mỗi cột tự động dựa trên kiểu dữ liệu của cột, vì vậy bạn không cần phải lo lắng về việc chọn thuật toán nén phù hợp cho từng cột. Để đảm bảo Redshift thực hiện phân tích nén, chúng tôi sẽ đặt tham số **COMPUPDATE** thành **PRESET** trong các lệnh **COPY**.

Dữ liệu mẫu để tải đã được cung cấp trong một  [Amazon S3](https://us-west-2.console.aws.amazon.com/s3/buckets/redshift-immersionday-labs?region=us-west-2&prefix=data%2F&showversions=false&bucketType=general) công khai.

Hãy sao chép các câu lệnh dưới đây để tải dữ liệu vào 7 bảng.

```jsx
COPY region FROM 's3://redshift-immersionday-labs/data/region/region.tbl.lzo'
iam_role default
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy customer from 's3://redshift-immersionday-labs/data/customer/customer.tbl.'
iam_role default
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy orders from 's3://redshift-immersionday-labs/data/orders/orders.tbl.'
iam_role default
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy part from 's3://redshift-immersionday-labs/data/part/part.tbl.'
iam_role default
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy supplier from 's3://redshift-immersionday-labs/data/supplier/supplier.json' manifest
iam_role default
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy lineitem from 's3://redshift-immersionday-labs/data/lineitem-part/'
iam_role default
region 'us-west-2' gzip delimiter '|' COMPUPDATE PRESET;

copy partsupp from 's3://redshift-immersionday-labs/data/partsupp/partsupp.tbl.'
iam_role default
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;
```

![image.png](/images/2/2-4.png)

Thời gian ước tính để tải dữ liệu như sau. Trong khi chờ quá trình sao chép kết thúc, bạn có thể chuyển sang phần tiếp theo  [Tải dữ liệu từ máy cục bộ bằng QEV2](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab2#loading-data-from-your-local-machine-using-qev2)

- REGION (5 rows) - 2s
- CUSTOMER (15M rows) – 2m
- ORDERS - (76M rows) - 10s
- PART - (20M rows) - 2m
- SUPPLIER - (1M rows) - 10s
- LINEITEM - (303M rows) - 22s
- PARTSUPPLIER - (80M rows) - 15s

> **Lưu ý**: Một số điểm chính rút ra từ câu lệnh COPY ở trên.

- **COMPUPDATE PRESET ON** sẽ gán mã hóa nén theo các thực tiễn tốt nhất của Amazon Redshift liên quan đến kiểu dữ liệu của cột nhưng không phân tích dữ liệu trong bảng.
- Lệnh **COPY** cho bảng **REGION** chỉ định một tệp cụ thể (`region.tbl.lzo`), trong khi lệnh **COPY** cho các bảng khác chỉ định một tiền tố cho nhiều tệp (`lineitem.tbl.`).
- Lệnh **COPY** cho bảng **SUPPLIER** chỉ định một tệp manifest (`supplier.json`). Một tệp manifest là một tệp JSON liệt kê các tệp cần được tải và vị trí của chúng.

- **Nạp dữ liệu từ máy tính cá nhân sử dụng QEV2**

Bạn sẽ nạp bảng *nation* thông qua *Query Editor v2* bằng cách nhập dữ liệu từ máy tính cá nhân của bạn.

*+ Thiết lập một lần*

Trong quá trình thiết lập một lần, bạn cần tạo một bucket S3 trong cùng khu vực với phiên bản Redshift. Sau đó, thêm đường dẫn bucket S3 vào trang cài đặt tài khoản của *Query Editor v2*.

Bucket S3 đã được tạo sẵn trong môi trường lab này. Điều hướng đến [AWS CloudFormation](https://ap-southeast-2.signin.aws.amazon.com/oauth?response_type=code&client_id=arn%3Aaws%3Asignin%3A%3A%3Aconsole%2Fcloudformation&redirect_uri=https%3A%2F%2Fconsole.aws.amazon.com%2Fcloudformation%2Fhome%3FhashArgs%3D%2523%26isauthcode%3Dtrue%26oauthStart%3D1723994246570%26state%3DhashArgsFromTB_ap-southeast-2_4edb45c164185361&forceMobileLayout=0&forceMobileApp=0&code_challenge=9xZ7-RKc3ZqaBRZs2hRfaMaNN99JIO-vOWYdGlBdM6M&code_challenge_method=SHA-256) và nhấp vào stack 'cfn'. Trong tab 'Outputs', ghi lại giá trị của S3Bucket. Bạn sẽ cần giá trị này trong bước tiếp theo.

![image.png](/images/2/2-55.png)

Trong *Query Editor v2* (QEV2), đi đến **Settings** (góc dưới bên trái) → **Account settings** → **S3 bucket name**.

![image.png](/images/2/2-61.png)

![image.png](/images/2/2-72.png)

*+ Nạp tập tin*

Bạn có thể tải xuống tệp mẫu [*Nation.csv*](https://redshift-immersionday-labs.s3.us-west-2.amazonaws.com/data/nation/Nation.csv) về máy tính cá nhân của mình bằng cách sử dụng liên kết dưới đây.

Hãy nạp tệp *Nation* đã tải xuống trên máy tính của bạn vào Redshift.

1. Từ phần **resources** (tài nguyên), chọn và kết nối với nhóm công việc serverless của bạn.

> **Chú ý:**
> Bảng *Nation* đã được tạo trong bước  [Create Tables](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab2#create-tables). Tuy nhiên, bạn cũng có thể tạo bảng thông qua giao diện người dùng bằng cách thực hiện theo các bước sau: **Create**-->**Table** trong bảng điều hướng bên trái của *Query Editor v2*.

2. Nhấp vào **Load data** -- chọn **Load from local file**  --Duyệt và chọn Nation.csv file đã tải xuống trước đó -- **Data conversion parameters** -- **Check Ignore header rows** -- **Next** -- **Table options** chọn nhóm công việc serverless của bạn -- **Database** = dev -- **Schema** = public -- **Table** = nation -- **Load data** -- Thông báo **Loading data from a local file succeeded**  sẽ hiển thị ở phía trên cùng.

![image.png](/images/2/2-82.png)

> **Chú ý:** **Máy tính có hạn chế tải lên**
> 
> Nếu máy tính của bạn có các hạn chế không cho phép tải lên S3, bạn sẽ không thể nạp tệp cục bộ. Tuy nhiên, bạn có thể sử dụng *Query Editor v2* để nạp tệp trực tiếp từ S3. Để thực hiện điều này, hãy chọn tùy chọn **Load from S3 bucket** và nhập vị trí sau:
> 
> `s3://redshift-immersionday-labs/data/nation/Nation.csv`
> 
> Bạn cần chọn vị trí S3 là `us-west-2`.
> 
> Làm theo các hướng dẫn dưới đây như đã nêu. Thay đổi duy nhất là bạn sẽ cần chọn vai trò IAM khi chọn bảng để nạp. Sẽ chỉ có một vai trò khả dụng trong **AWS Sponsored Events** (`RedshiftServerlessImmersionRole`).

![image.png](/images/2/2-91.png)

![image.png](/images/2/2-101.png)

### **2.4 Kiểm tra độ chính xác dữ liệu**

Hãy thực hiện một kiểm tra nhanh về số lượng trên một vài bảng để đảm bảo rằng dữ liệu đã được tải lên như mong đợi. Bạn có thể mở một trình soạn thảo mới và chạy các truy vấn dưới đây nếu quá trình sao chép vẫn đang tải một số bảng.

```jsx
 --Number of rows= 5
select count(*) from region;

 --Number of rows= 25
select count(*) from nation;

 --Number of rows= 76,000,000
select count(*) from orders;
```

![image.png](/images/2/2-11.png)

![image.png](/images/2/2-12.png)

![image.png](/images/2/2-13.png)

### **2.5 Khắc phục sự cố tải dữ liệu**

Để khắc phục các vấn đề liên quan đến việc tải dữ liệu, bạn có thể truy vấn bảng `SYS_LOAD_ERROR_DETAIL`.

Ngoài ra, bạn có thể xác nhận dữ liệu của mình mà không cần thực sự tải bảng. Bạn có thể sử dụng tùy chọn `NOLOAD` với lệnh `COPY` để đảm bảo rằng dữ liệu của bạn sẽ tải mà không gặp lỗi trước khi thực hiện việc tải dữ liệu thực tế. Lưu ý rằng chạy `COPY` với tùy chọn `NOLOAD` sẽ nhanh hơn nhiều so với việc tải dữ liệu vì nó chỉ phân tích cú pháp các tệp.

Hãy thử tải bảng `CUSTOMER` với một tệp dữ liệu khác có các cột không khớp.

```jsx
COPY customer FROM 's3://redshift-immersionday-labs/data/nation/nation.tbl.'
iam_role default
region 'us-west-2' lzop delimiter '|' noload;
```

![image.png](/images/2/2-14.png)

Bạn sẽ nhận được lỗi sau.

```jsx
ERROR: Load into table 'customer' failed. Check 'sys_load_error_detail' system table for details.
```

Truy vấn bảng hệ thống STL_LOAD_ERROR để biết chi tiết.

```jsx
select * from SYS_LOAD_ERROR_DETAIL;
```

![image.png](/images/2/2-15.png)

Lưu ý rằng có một hàng cho cột c_nationkey có kiểu dữ liệu int với thông báo lỗi "Chữ số không hợp lệ, Giá trị 'h', Pos 1" cho biết rằng bạn đang cố tải ký tự vào một cột số nguyên.

### **2.6 Bảo trì bảng tự động - ANALYZE và VACUUM**

- **Phân Tích (ANALYZE):**

Khi tải dữ liệu vào một bảng trống, lệnh `COPY` theo mặc định sẽ thu thập các thống kê (ANALYZE). Nếu bạn đang tải dữ liệu vào một bảng không trống bằng lệnh `COPY`, trong hầu hết các trường hợp, bạn không cần phải chạy lệnh `ANALYZE` một cách rõ ràng. Amazon Redshift theo dõi các thay đổi trong khối lượng công việc của bạn và tự động cập nhật các thống kê trong nền. Để giảm thiểu tác động đến hiệu suất hệ thống của bạn, quá trình phân tích tự động sẽ chạy trong các khoảng thời gian khi khối lượng công việc nhẹ.

Nếu bạn cần phân tích bảng ngay sau khi tải, bạn vẫn có thể chạy lệnh `ANALYZE` thủ công.

Để chạy lệnh `ANALYZE` trên bảng `orders`, sao chép lệnh sau đây và chạy nó.

```jsx
ANALYZE orders;
```

![image.png](/images/2/2-16.png)

- **Vacuum:**

**+ Vacuum Delete:** Khi bạn thực hiện lệnh xóa trên một bảng, các hàng sẽ được đánh dấu để xóa (xóa mềm), nhưng không bị loại bỏ. Khi bạn thực hiện cập nhật, các hàng hiện có sẽ được đánh dấu để xóa (xóa mềm) và các hàng được cập nhật sẽ được chèn vào như các hàng mới. Amazon Redshift tự động chạy một hoạt động `VACUUM DELETE` trong nền để thu hồi không gian đĩa bị chiếm bởi các hàng đã được đánh dấu để xóa bởi các hoạt động `UPDATE` và `DELETE`, đồng thời nén bảng để giải phóng không gian bị tiêu tốn. Để giảm thiểu tác động đến hiệu suất hệ thống của bạn, quá trình `VACUUM DELETE` tự động sẽ chạy trong các khoảng thời gian khi khối lượng công việc nhẹ.

Nếu bạn cần thu hồi không gian đĩa ngay sau một hoạt động xóa lớn, ví dụ như sau khi tải dữ liệu lớn, bạn vẫn có thể chạy lệnh `VACUUM DELETE` thủ công. Hãy xem cách `VACUUM DELETE` thu hồi không gian bảng sau khi thực hiện thao tác xóa.

Trước tiên, hãy ghi lại `tbl_rows` (Tổng số hàng trong bảng. Giá trị này bao gồm các hàng đã được đánh dấu để xóa, nhưng chưa được làm sạch bằng VACUUM) và `estimated_visible_rows` (Số hàng hiển thị ước tính trong bảng. Giá trị này không bao gồm các hàng đã được đánh dấu để xóa) cho bảng `ORDERS`. Sao chép lệnh sau đây và chạy nó.

```jsx
select "table", size, tbl_rows, estimated_visible_rows
from SVV_TABLE_INFO
where "table" = 'orders';
```

![image.png](/images/2/2-17.png)

Tiếp theo, xóa các hàng khỏi bảng ORDERS. Sao chép lệnh sau và chạy nó.

```jsx
delete orders where o_orderdate between '1997-01-01' and '1998-01-01';
```

![image.png](/images/2/2-18.png)

Tiếp theo, ghi lại tbl_rows và Estimate_visible_rows cho bảng ORDERS sau khi xóa.

Sao chép lệnh sau và chạy nó. Lưu ý rằng giá trị tbl_rows không thay đổi sau khi xóa. Điều này là do các hàng được đánh dấu để xóa mềm nhưng VACUUM DELETE chưa được chạy để lấy lại dung lượng.

```jsx
select "table", size, tbl_rows, estimated_visible_rows
from SVV_TABLE_INFO
where "table" = 'orders';
```

![image.png](/images/2/2-19.png)

Bây giờ, hãy chạy lệnh VACUUM DELETE. Sao chép lệnh sau và chạy nó.

```jsx
vacuum delete only orders;
```

![image.png](/images/2/2-20.png)

Xác nhận rằng lệnh VACUUM đã lấy lại dung lượng bằng cách chạy lại truy vấn sau và lưu ý rằng giá trị tbl_rows đã thay đổi.

```jsx
select "table", size, tbl_rows, estimated_visible_rows
from SVV_TABLE_INFO
where "table" = 'orders';
```

![image.png](/images/2/2-21.png)

**+ Vacuum Sort:**

Khi bạn định nghĩa các **SORT KEYS** trên bảng của mình, Amazon Redshift tự động sắp xếp dữ liệu trong nền để duy trì dữ liệu bảng theo thứ tự của khóa sắp xếp. Việc có các khóa sắp xếp trên các cột được lọc thường xuyên làm cho việc loại bỏ các khối dữ liệu, vốn đã hiệu quả trong Amazon Redshift, trở nên hiệu quả hơn.

Amazon Redshift theo dõi các truy vấn quét của bạn để xác định các phần nào của bảng sẽ được hưởng lợi từ việc sắp xếp và tự động chạy `VACUUM SORT` để duy trì thứ tự của khóa sắp xếp. Để giảm thiểu tác động đến hiệu suất hệ thống của bạn, quá trình `VACUUM SORT` tự động sẽ chạy trong các khoảng thời gian khi khối lượng công việc nhẹ.

Lệnh `COPY` tự động sắp xếp và tải dữ liệu theo thứ tự của khóa sắp xếp. Do đó, nếu bạn đang tải một bảng trống bằng lệnh `COPY`, dữ liệu đã được sắp xếp theo thứ tự của khóa sắp xếp. Nếu bạn đang tải dữ liệu vào một bảng không trống bằng lệnh `COPY`, bạn có thể tối ưu hóa quá trình tải bằng cách tải dữ liệu theo thứ tự tăng dần của khóa sắp xếp vì `VACUUM SORT` sẽ không cần thiết khi dữ liệu của bạn đã ở trong thứ tự khóa sắp xếp.

Ví dụ, `orderdate` là khóa sắp xếp trên bảng `orders`. Nếu bạn luôn tải dữ liệu vào bảng `orders` với `orderdate` là ngày hiện tại, vì ngày hiện tại luôn tăng dần, dữ liệu sẽ luôn được tải theo thứ tự tăng dần của khóa sắp xếp (`orderdate`). Do đó, trong trường hợp này, `VACUUM SORT` sẽ không cần thiết cho bảng `orders`.

Nếu bạn cần chạy `VACUUM SORT`, bạn vẫn có thể chạy thủ công như được chỉ ra dưới đây. Sao chép lệnh sau và chạy nó.

```jsx
vacuum sort only orders;
```

![image.png](/images/2/2-22.png)

**+ Vacuum Recluster:**

Hãy sử dụng `VACUUM recluster` bất cứ khi nào có thể cho các hoạt động `VACUUM` thủ công. Điều này đặc biệt quan trọng đối với các đối tượng lớn có tần suất nạp dữ liệu cao và các truy vấn chỉ truy cập vào dữ liệu mới nhất. `VACUUM recluster` chỉ sắp xếp lại các phần của bảng chưa được sắp xếp, do đó chạy nhanh hơn. Các phần của bảng đã được sắp xếp sẽ được giữ nguyên. Lệnh này không gộp dữ liệu mới được sắp xếp với vùng đã được sắp xếp. Nó cũng không thu hồi toàn bộ không gian đã được đánh dấu để xóa.

Để chạy `VACUUM recluster` trên bảng `orders`, sao chép lệnh sau và chạy nó.

```jsx
vacuum recluster orders;
```

![image.png](/images/2/2-23.png)

**+ Vacuum Boost:**

`Boost` chạy lệnh `VACUUM` với các tài nguyên tính toán bổ sung khi chúng có sẵn. Với tùy chọn `BOOST`, `VACUUM` hoạt động trong một cửa sổ và chặn các thao tác xóa và cập nhật đồng thời trong suốt quá trình `VACUUM`. Lưu ý rằng chạy `VACUUM` với tùy chọn `BOOST` sẽ cạnh tranh tài nguyên hệ thống, điều này có thể ảnh hưởng đến hiệu suất của các truy vấn khác. Do đó, nên chạy `VACUUM BOOST` khi tải trên hệ thống nhẹ, chẳng hạn như trong quá trình bảo trì.

Để chạy `VACUUM recluster` trên bảng `orders` trong chế độ `boost`, sao chép lệnh sau và chạy nó.

```jsx
vacuum recluster orders boost;
```

![image.png](/images/2/2-24.png)

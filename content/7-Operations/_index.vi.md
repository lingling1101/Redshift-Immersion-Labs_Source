+++
title = "Các hoạt động"
date = 2021
weight = 7
chapter = false
pre = "<b>7. </b>"
+++

**7. Các hoạt động**

Trong bài thực hành này, bạn sẽ thực hiện việc giám sát (từ cả bảng điều khiển và các chế độ xem hệ thống), tính năng ghi nhật ký kiểm toán, thay đổi cài đặt và giới hạn RPU cơ bản.

**Nội dung**

- Trước khi bắt đầu
- Giám sát
- Chế độ xem hệ thống
- Ghi nhật ký kiểm toán
- Thay đổi RPU và đặt giới hạn

**7.1 Trước khi bắt đầu**

Bài thực hành này giả định rằng bạn đã khởi tạo một điểm cuối **Amazon Redshift Serverless**. Nếu bạn chưa thực hiện, vui lòng xem phần  [Bắt Đầu](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab1) và làm theo hướng dẫn ở đó. Chúng tôi sẽ sử dụng **Amazon Redshift [QueryEditorV2](https://console.aws.amazon.com/sqlworkbench/home)** và bảng điều khiển [**Amazon Redshift Serverless**](https://console.aws.amazon.com/redshiftv2/home?#serverless-dashboard) cho bài thực hành này.

**7.2 Giám sát**

Trong phần này, bạn sẽ trải nghiệm các tùy chọn giám sát khác nhau và thiết lập bộ lọc để trải nghiệm chức năng. Bạn sẽ sử dụng bảng điều khiển để xem lịch sử truy vấn, hiệu suất cơ sở dữ liệu và việc sử dụng tài nguyên.

Khi bạn đã vào bảng điều khiển **Redshift Serverless**, điều hướng đến phần [**Query and database monitoring**](https://console.aws.amazon.com/redshiftv2/home?#serverless-query-and-database-monitoring) (Giám sát truy vấn và cơ sở dữ liệu) ở menu bên trái.

- Chọn nhóm công việc **workgroup-xxxxxxx**
- Mở rộng **Additional filtering options** (Tùy chọn lọc bổ sung). Bạn có thể lọc các chỉ số này theo nhiều danh mục khác nhau. Cảm thấy tự do thay đổi các giá trị này dựa trên các mẫu khối lượng công việc mà bạn quan tâm nhất. Đối với bài thực hành này, hãy đặt các giá trị như trong ảnh chụp màn hình - đây là các giá trị mặc định để bắt đầu.

![image.png](/images/7/7-1.png)

Bạn sẽ không thể nhìn thấy các truy vấn. Bạn cần cấp quyền truy cập giám sát cho vai trò bảng điều khiển của mình. Đây là bước tiên quyết.

Đăng nhập với tư cách siêu người dùng (awsuser) trong Trình soạn thảo truy vấn và chạy bên dưới SQL.

```jsx

-- Run below command if you are running this event from workshop
grant role sys:monitor to "IAMR:WSParticipantRole";

-- Run below command if you are running this event from Event Engine
grant role sys:monitor to "IAMR:TeamRole" 
```

- Bây giờ hãy quay lại phần [**Query and database monitoring**](https://console.aws.amazon.com/redshiftv2/home?#serverless-query-and-database-monitoring) (Giám sát truy vấn và cơ sở dữ liệu) ở menu bên trái và thay đổi một số bộ lọc để bắt đầu xem các truy vấn.
- Cuộn xuống dưới, bạn có thể thấy thời gian chạy truy vấn của bạn trong khoảng thời gian đã chọn. Sử dụng biểu đồ này để xem xét độ đồng thời của các truy vấn cũng như nghiên cứu thêm về những truy vấn mất nhiều thời gian để thực thi hơn mong đợi.

![image.png](/images/7/7-2.png)

- Cuộn lại bên dưới để xem biểu đồ Truy vấn và tải. Tại đây bạn có thể thấy tất cả các truy vấn đã hoàn thành, đang chạy và bị hủy bỏ.

![image.png](/images/7/7-3.png)

Đi đến tab **Database Performance** (Hiệu suất cơ sở dữ liệu) để xem:

- **Queries completed per second** (Số truy vấn hoàn thành mỗi giây): Số lượng truy vấn trung bình hoàn thành mỗi giây.

![image.png](/images/7/7-4.png)

- **Queries duration** (Thời gian truy vấn): Thời gian trung bình để hoàn thành một truy vấn.

![image.png](/images/7/7-5.png)

- **Database connections** (Kết nối cơ sở dữ liệu): Số lượng kết nối cơ sở dữ liệu hoạt động trung bình.

![image.png](/images/7/7-6.png)

- **Running and Queued queries** (Truy vấn đang chạy và đang xếp hàng)

![image.png](/images/7/7-7.png)

Đi đến phần **Resource Monitoring** (Giám sát tài nguyên) trên thanh điều hướng bên trái.

- Chọn **default workgroup** (nhóm công việc mặc định)
- Mở rộng **Additional filtering options** (Tùy chọn lọc bổ sung). Chọn khoảng thời gian **1 phút** và xem xét kết quả. Bạn cũng có thể thử các khoảng thời gian khác để xem kết quả.

![image.png](/images/7/7-8.png)

- **RPU Capacity Used** (Công suất RPU đã sử dụng) - Số lượng RPUs đã tiêu thụ.

![image.png](/images/7/7-9.png)

- **Compute usage** (Sử dụng tính toán) - Số giây RPU.

![image.png](/images/7/7-10.png)

**7.3 Chế độ xem hệ thống**

Dưới đây là các chế độ xem hệ thống trong **Amazon Redshift Serverless** được sử dụng để giám sát truy vấn và sử dụng khối lượng công việc. Các chế độ xem này nằm trong lược đồ **pg_catalog**. Các chế độ xem hệ thống này đã được thiết kế để cung cấp thông tin cần thiết để giám sát Amazon Redshift Serverless, đơn giản hơn nhiều so với các chế độ xem hệ thống khác có sẵn cho các cluster đã cấp phát.

- SYS_QUERY_HISTORY
- SYS_QUERY_DETAIL
- SYS_EXTERNAL_QUERY_DETAIL
- SYS_LOAD_HISTORY
- SYS_LOAD_ERROR_DETAIL
- SYS_UNLOAD_HISTORY
- SYS_SERVERLESS_USAGE

Vui lòng tham khảo các bổ sung mới tại [https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-monitoring.html](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-monitoring.html)

Chúng ta sẽ đi qua một số truy vấn giám sát hệ thống thường dùng.

- Chạy truy vấn dưới đây để tìm 10 truy vấn gần đây đã hoàn thành, đang chạy và đang xếp hàng.

```jsx
SELECT user_id,
       query_id,
       transaction_id,
       session_id,
       status,
       trim(database_name) AS database_name,
       start_time,
       end_time,
       result_cache_hit,
       elapsed_time,
       queue_time,
       execution_time
FROM sys_query_history
WHERE status IN ('success','running','queued')
ORDER BY start_time
LIMIT 10;
```

![image.png](/images/7/7-11.png)

- Chạy truy vấn dưới đây để hiển thị tóm tắt việc sử dụng serverless, bao gồm lượng tài nguyên tính toán được sử dụng để xử lý các truy vấn và lượng **Amazon Redshift managed storage** được sử dụng

```jsx
SELECT start_time,
        end_time,
        compute_seconds,
        compute_capacity,
        data_storage
FROM sys_serverless_usage
ORDER BY start_time desc;
```

![image.png](/images/7/7-12.png)

Trong kết quả, bạn có thể thấy các khoảng thời gian không hoạt động khi cluster tự động dừng và tự động khôi phục khi các truy vấn bắt đầu xuất hiện. Bạn sẽ không bị tính phí khi cluster đang ở trạng thái dừng.

Truy vấn để hiển thị số lượng hàng đã tải, số byte, số bảng và nguồn dữ liệu của các lệnh COPY

```jsx
SELECT query_id,
       table_name,
       data_source,
       loaded_rows,
       loaded_bytes
FROM sys_load_history
ORDER BY query_id DESC;
```

![image.png](/images/7/7-13.png)

**7.4 Ghi nhật ký kiểm toán**

Bạn có thể cấu hình Amazon Redshift Serverless để xuất dữ liệu kết nối, người dùng và hoạt động của người dùng vào một nhóm nhật ký trong Amazon CloudWatch Logs khi tạo các nhóm công việc.

Bước đầu tiên trong quy trình là đảm bảo rằng ghi nhật ký kiểm toán đã được bật cho nhóm công việc.

- Đi đến **Namespace** → **security and encryption**. Xác minh rằng ghi nhật ký kiểm toán đã được bật.

![image.png](/images/7/7-14.png)

![image.png](/images/7/7-15.png)

- Chọn tùy chọn **Edit** nếu ghi nhật ký kiểm toán bị tắt và đánh dấu tất cả 3 tùy chọn như hình dưới đây.

![image.png](/images/7/7-16.png)

- Bây giờ hãy chạy các lệnh sau trong **query editor** kết nối đến nhóm công việc mặc định của serverless.

```jsx
create table orders_new (
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
```

![image.png](/images/7/7-17.png)

```jsx
copy orders_new from 's3://redshift-immersionday-labs/data/orders/orders.tbl.'
iam_role default
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

select * from orders_new limit 100;
```

![image.png](/images/7/7-18.png)

![image.png](/images/7/7-19.png)

Bạn có thể giám sát các sự kiện trong Amazon CloudWatch Logs.

- Đi đến dịch vụ **Amazon CloudWatch** và chọn **Metrics** → **All metrics** trên menu điều hướng bên trái.
- Chọn **AWS/Redshift-Serverless** để lấy chi tiết về việc sử dụng Serverless.

![image.png](/images/7/7-20.png)

Các chỉ số Amazon Redshift Serverless được chia thành các nhóm chỉ số tính toán, dữ liệu và lưu trữ. Chọn **Workgroup*** để lấy chi tiết về số giây tính toán.

![image.png](/images/7/7-21.png)

Chọn nhóm công việc của bạn để lấy chi tiết về tài nguyên tính toán cho nhóm công việc cụ thể. 


![image.png](/images/7/7-22.png)

Chọn **DatabaseName**, **Namespace** để lấy chi tiết về tài nguyên lưu trữ cho một namespace.

![image.png](/images/7/7-23.png)

Chọn cơ sở dữ liệu **Dev** cho namespace của bạn để tìm tổng số bảng như hình dưới đây. 

![image.png](/images/7/7-24.png)

Bạn cũng có thể xuất các sự kiện nhóm nhật ký CloudWatch sang S3. 

**LƯU Ý**: Bạn có thể không truy cập được điều này trong môi trường thực hành. Nhưng bạn có thể thử trong môi trường của riêng bạn.

![image.png](/images/7/7-25.png)

**7.5 Thay đổi RPUs và đặt giới hạn**

- Chọn default workgroup của bạn, nhấp vào tab **Limits** (Giới hạn).
- Nhấp vào **Edit** (Chỉnh sửa) trong phần **Base capacity in Redshift processing units (RPUs)** (Công suất cơ bản bằng đơn vị xử lý Redshift - RPU).

![image.png](/images/7/7-26.png)

Mặc định, khi bạn khởi tạo điểm cuối serverless, bạn sẽ được cấp 128 RPUs. Ở đây, bạn có thể điều chỉnh cài đặt **Base capacity** (Công suất cơ bản) từ 32 RPUs đến 512 RPUs theo các đơn vị gia tăng 8 RPUs. Mỗi RPU là một đơn vị tài nguyên tính toán với 16 GB bộ nhớ.

Đối với bài thực hành này, bạn có thể giữ nguyên ở 128 RPUs. Chọn **Cancel** (Hủy).

- Cuộn xuống **Usage limits** (Giới hạn sử dụng) và nhấp vào **Manage usage limits** (Quản lý giới hạn sử dụng).

![image.png](/images/7/7-27.png)

- Nhấp vào **Add limit** (Thêm giới hạn).

![image.png](/images/7/7-28.png)

Trong tab này, bạn có thể cấu hình giới hạn công suất sử dụng để kiểm soát hóa đơn Redshift Serverless của bạn. Để kiểm soát việc sử dụng, hãy đặt số giờ RPU tối đa theo tần suất.

- Đặt **Frequency** (Tần suất) thành **Daily** (Hàng ngày), **Usage limit (hours)** (Giới hạn sử dụng (giờ)) thành **1**, và **Action** (Hành động) thành **Turn off user queries** (Tắt truy vấn của người dùng).

(Có thể) Bạn có thể đặt cảnh báo qua email bằng cách chọn **Create SNS topic** (Tạo chủ đề SNS) để nhận thông báo khi đạt đến giới hạn sử dụng. 

- Sau khi hoàn tất, nhấp vào **Save changes** (Lưu thay đổi). Sau 1 giờ RPU sử dụng hàng ngày, người dùng của bạn sẽ không còn gửi truy vấn đến Redshift Serverless và sẽ dừng việc tính phí RPU.

![image.png](/images/7/7-29.png)
- Chạy truy vấn mẫu dưới đây vài lần. Sau vài lần, bạn sẽ nhận được thông báo lỗi giới hạn sử dụng.

```jsx
select n_name, s_name, l_shipmode,
  SUM(L_QUANTITY) Total_Qty
from lineitem
join supplier on l_suppkey = s_suppkey
join nation on s_nationkey = n_nationkey
where datepart(year, L_SHIPDATE) > 1997  
group by 1,2,3
order by 3 desc;
```

![image.png](/images/7/7-30.png)

- Bạn sẽ nhận được lỗi: `ERROR: Query reached usage limit*` khi đạt đến giới hạn và một thông báo qua email sẽ được gửi nếu bạn đã đăng ký qua chủ đề SNS.

![image.png](/images/7/7-31.png)

- Quay lại **Manage usage limits** (Quản lý giới hạn sử dụng) và xóa giới hạn trước khi tiếp tục bài thực hành tiếp theo. Nếu bạn bỏ qua bước này, bạn sẽ không thể thực hiện thêm các truy vấn.

+++
title = "Truy vấn Data Lake - Redshift Spectrum"
date = 2021
weight = 6
chapter = false
pre = "<b>6. </b>"
+++

**VI. Truy vấn Data Lake - Redshift Spectrum**

Trong bài thực hành này, chúng tôi sẽ hướng dẫn bạn cách truy vấn dữ liệu trong Amazon S3 Data Lake bằng Amazon Redshift mà không cần tải hoặc di chuyển dữ liệu. Chúng tôi cũng sẽ trình bày cách bạn có thể sử dụng các view để kết hợp dữ liệu trong Redshift Managed storage với dữ liệu trong S3. Bạn có thể truy vấn dữ liệu có cấu trúc và bán cấu trúc từ các tệp trong Amazon S3 mà không cần sao chép hoặc di chuyển dữ liệu vào các bảng của Amazon Redshift. 

![image.png](/images/6/6-1.png)

**Nội dung**

- Trước khi bắt đầu
- Mô tả trường hợp sử dụng
- Hướng dẫn
- Trước khi kết thúc

**6.1 Trước khi bắt đầu**

Bài thực hành này giả định rằng bạn đã khởi tạo một Redshift Serverless Warehouse. Nếu bạn chưa tạo Redshift Serverless Warehouse, hãy xem phần [Bắt Đầu](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab1). Chúng tôi sẽ sử dụng Redshift Query Editor V2 trong bảng điều khiển Redshift cho bài thực hành này.

Vui lòng tìm khu vực của bạn bằng cách làm theo hình ảnh bên dưới và chọn bộ dữ liệu s3 theo hướng dẫn cho khu vực của bạn.

**6.2 Mô tả trường hợp sử dụng**

**Mục tiêu**: Rút ra những thông tin dữ liệu để thể hiện tác động của trận bão tuyết đến số lượng chuyến xe taxi vào tháng 1 năm 2016.

**Mô tả tập dữ liệu**: Dữ liệu chuyến đi taxi tại thành phố New York bao gồm số lượng chuyến đi taxi theo năm và tháng cho 3 công ty taxi khác nhau - fhv, green, và yellow.

**Vị trí tập dữ liệu trên S3**:

+ Khu vực **us-east-1** - [https://s3.console.aws.amazon.com/s3/buckets/redshift-demos?region=us-east-1&prefix=data/NY-Pub/](https://s3.console.aws.amazon.com/s3/buckets/redshift-demos?region=us-east-1&prefix=data/NY-Pub/)
+ Khu vực **us-west-2** - [https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics?prefix=canonical/NY-Pub/](https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics?prefix=canonical/NY-Pub/)
+ Khu vực **eu-west-1** - [https://s3.console.aws.amazon.com/s3/buckets/redshift-demos-dub?region=eu-west-1&prefix=NY-Pub/](https://s3.console.aws.amazon.com/s3/buckets/redshift-demos-dub?region=eu-west-1&prefix=NY-Pub/)
+ Khu vực **ap-northeast-1** - [https://s3.console.aws.amazon.com/s3/buckets/redshift-demos-nrt?region=ap-northeast-1&prefix=NY-Pub/](https://s3.console.aws.amazon.com/s3/buckets/redshift-demos-nrt?region=ap-northeast-1&prefix=NY-Pub/)

Dưới đây là tổng quan về các bước trong trường hợp sử dụng này liên quan đến bài thực hành.
![image.png](/images/6/6-2.png)

**6.3 Hướng dẫn**

**1. Tạo và chạy Glue crawler để điền vào Glue data catalog**

Trong phần này của bài thực hành, chúng ta sẽ thực hiện các hoạt động sau:
- Truy vấn dữ liệu lịch sử lưu trữ trên S3 bằng cách tạo một cơ sở dữ liệu (DB) ngoài cho Redshift Spectrum.
- Xem xét kỹ dữ liệu lịch sử, có thể tổng hợp dữ liệu theo những cách mới để xem các xu hướng theo thời gian hoặc các khía cạnh khác.
- Lưu ý rằng lược đồ phân vùng là Năm, Tháng, Loại (trong đó Loại là công ty taxi). Dưới đây là ảnh chụp màn hình: [https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics/canonical/NY-Pub/](https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics/canonical/NY-Pub/)

![image.png](/images/6/6-3.png)

**Tạo lược đồ (và cơ sở dữ liệu) ngoài cho Redshift Spectrum**

Bạn có thể tạo một bảng ngoài trong Amazon Redshift, AWS Glue, Amazon Athena, hoặc một Apache Hive metastore. Nếu bảng ngoài của bạn được định nghĩa trong AWS Glue, Athena, hoặc một Hive metastore, trước tiên bạn cần tạo một lược đồ ngoài tham chiếu đến cơ sở dữ liệu ngoài. Sau đó, bạn có thể tham chiếu bảng ngoài trong câu lệnh SELECT của mình bằng cách thêm tiền tố tên bảng với tên lược đồ mà không cần phải tạo bảng trong Amazon Redshift.

Trong bài thực hành này, bạn sẽ sử dụng AWS Glue Crawler để tạo bảng ngoài **adb305.ny_pub** được lưu trữ dưới định dạng **parquet** trong vị trí S3 của khu vực tương ứng.

**+ Điều hướng đến Trang Glue Crawler:**

- Khu vực **us-east-1** - [https://us-east-1.console.aws.amazon.com/glue/home](https://us-east-1.console.aws.amazon.com/glue/home)
- Khu vực **us-west-2** - [https://us-west-2.console.aws.amazon.com/glue/home](https://us-west-2.console.aws.amazon.com/glue/home)
- Khu vực **eu-west-1** - [https://eu-west-1.console.aws.amazon.com/glue/home](https://eu-west-1.console.aws.amazon.com/glue/home)
- Khu vực **ap-northeast-1** - [https://ap-northeast-1.console.aws.amazon.com/glue/home](https://ap-northeast-1.console.aws.amazon.com/glue/home)

![image.png](/images/6/6-4.png)

**+** Nhấp vào **Create Crawler**, nhập tên crawler là **NYTaxiCrawler** và nhấp **Next**.

![image.png](/images/6/6-5.png)

**+** Chọn **Add a data source.**

![image.png](/images/6/6-6.png)

**+** Chọn **S3** làm nguồn dữ liệu.

**+** Chọn **In a different account** (Trong một tài khoản khác)

**+** Nhập đường dẫn tệp S3:

- Đối với khu vực **us-east-1** - `s3://redshift-demos/data/NY-Pub/`
- Đối với khu vực **us-west-2** - `s3://us-west-2.serverless-analytics/canonical/NY-Pub/`
- Đối với khu vực **eu-west-1** - `s3://redshift-demos-dub/NY-Pub/`
- Đối với khu vực **ap-northeast-1** - `s3://redshift-demos-nrt/NY-Pub/`

**+** Nhấp vào **Add an S3 data source** (Thêm một nguồn dữ liệu S3).

![image.png](/images/6/6-7.png)

**+** Nhấn **Next**

![image.png](/images/6/6-8.png)

**+** Nhấp vào **Create new IAM role** và nhấp **Next**

![image.png](/images/6/6-9.png)

**+** Nhập tên **AWSGlueServiceRole-RedshiftImmersion** và nhấp **Create**

![image.png](/images/6/6-10.png)

**+** Nhấp vào **Add database** và nhập tên là **spectrumdb**

![image.png](/images/6/6-11.png)

![image.png](/images/6/6-12.png)

**+** Quay lại **Glue Console**, làm mới cơ sở dữ liệu đích và chọn **spectrumdb**

![image.png](/images/6/6-13.png)

**+** Chọn tất cả các tùy chọn mặc định còn lại và nhấp **Create crawler**. Chọn crawler **NYTaxiCrawler** và nhấp **Run**.

![image.png](/images/6/6-14.png)

**+** Sau khi quá trình chạy crawler hoàn tất, bạn có thể thấy một bảng mới **ny_pub** trong **Glue Catalog**.

![image.png](/images/6/6-15.png)

**2. Tạo lược đồ ngoài adb305 trong Redshift và truy vấn từ bảng trong Glue catalog - ny_pub**

**+** Đi đến **Redshift console**:

- Khu vực **us-east-1** - [https://us-east-1.console.aws.amazon.com/redshiftv2/home?region=us-east-1#/serverless-setup](https://us-east-1.console.aws.amazon.com/redshiftv2/home?region=us-east-1#/serverless-setup)
- Khu vực **us-west-2** - [https://us-west-2.console.aws.amazon.com/redshiftv2/home?region=us-west-2#/serverless-setup](https://us-west-2.console.aws.amazon.com/redshiftv2/home?region=us-west-2#/serverless-setup)
- Khu vực **eu-west-1** - [https://eu-west-1.console.aws.amazon.com/redshiftv2/home?region=eu-west-1#/serverless-setup](https://eu-west-1.console.aws.amazon.com/redshiftv2/home?region=eu-west-1#/serverless-setup)
- Khu vực **ap-northeast-1** - [https://ap-northeast-1.console.aws.amazon.com/redshiftv2/home?region=ap-northeast-1#/serverless-setup](https://ap-northeast-1.console.aws.amazon.com/redshiftv2/home?region=ap-northeast-1#/serverless-setup)

Nhấp vào mục **Serverless dashboard** ở bên trái của bảng điều khiển. Nhấp vào không gian tên đã được cung cấp trước đó. Nhấp vào **Query data**.

![image.png](/images/6/6-16.png)

- Tạo một lược đồ ngoài **adb305** trỏ đến cơ sở dữ liệu **Glue Catalog** của bạn là **spectrumdb**.

```jsx
CREATE external SCHEMA adb305
FROM data catalog DATABASE 'spectrumdb'
IAM_ROLE default
CREATE external DATABASE if not exists;
```

![image.png](/images/6/6-17.png)

**Pin-point the Blizzard**

Bạn có thể truy vấn bảng **ny_pub**, được định nghĩa trong **Glue Catalog** từ lược đồ ngoài **Redshift**. Vào tháng 1 năm 2016, có một ngày có số lượng chuyến đi taxi thấp nhất do bão tuyết. Bạn có thể tìm ngày đó không?

```jsx
SELECT TO_CHAR(pickup_datetime, 'YYYY-MM-DD'),COUNT(*)
FROM adb305.ny_pub
WHERE YEAR = 2016 and Month = 01
GROUP BY 1
ORDER BY 2;
```

![image.png](/images/6/6-18.png)

**3. Tạo lược đồ nội bộ workshop_das**

Tạo một lược đồ **workshop_das** cho các bảng sẽ được lưu trữ trên **Redshift Managed Storage**.

```jsx
CREATE SCHEMA workshop_das;
```

![image.png](/images/6/6-19.png)

**4. Chạy CTAS để tạo và tải bảng Redshift workshop_das.taxi_201601 bằng cách chọn từ bảng ngoài**

Tạo bảng **workshop_das.taxi_201601** để tải dữ liệu cho công ty taxi **green** vào tháng 1 năm 2016

```jsx
CREATE TABLE workshop_das.taxi_201601 AS
SELECT *
FROM adb305.ny_pub
WHERE year = 2016 AND month = 1 AND type = 'green';
```

![image.png](/images/6/6-20.png)

**Lưu ý**: Còn về nén/ mã hóa cột thì sao? Hãy nhớ rằng trên một CTAS, Amazon Redshift tự động chỉ định mã hóa nén như sau:

- Các cột được định nghĩa là khóa sắp xếp được chỉ định mã hóa RAW.
- Các cột được định nghĩa là BOOLEAN, REAL, DOUBLE PRECISION, hoặc GEOMETRY được chỉ định mã hóa RAW.
- Các cột được định nghĩa là SMALLINT, INTEGER, BIGINT, DECIMAL, DATE, TIMESTAMP, hoặc TIMESTAMPTZ được chỉ định mã hóa AZ64.
- Các cột được định nghĩa là CHAR hoặc VARCHAR được chỉ định mã hóa LZO.
[https://docs.aws.amazon.com/redshift/latest/dg/r_CTAS_usage_notes.html](https://docs.aws.amazon.com/redshift/latest/dg/r_CTAS_usage_notes.html)

```jsx
ANALYZE COMPRESSION workshop_das.taxi_201601;
```

![image.png](/images/6/6-21.png)

- Thêm vào bảng **taxi_201601** với câu lệnh **INSERT/SELECT** cho các công ty taxi khác.

```jsx
INSERT INTO workshop_das.taxi_201601 (
SELECT *
FROM adb305.ny_pub
WHERE year = 2016 AND month = 1 AND type != 'green');
```

![image.png](/images/6/6-22.png)

**5. Xóa các phân vùng 201601 khỏi bảng ngoài**

Bây giờ chúng ta đã tải xong dữ liệu của tháng 1 năm 2016, chúng ta có thể xóa các phân vùng khỏi bảng Spectrum để không có sự trùng lặp giữa bảng **Redshift Managed Storage (RMS)** và bảng **Spectrum**.

```jsx
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='yellow');
```

![image.png](/images/6/6-23.png)

**6. Tạo view kết hợp public.adb305_view_NY_TaxiRides**

```jsx
CREATE VIEW adb305_view_NYTaxiRides AS
  SELECT * FROM workshop_das.taxi_201601
  UNION ALL
  SELECT * FROM adb305.ny_pub
WITH NO SCHEMA BINDING;
```

![image.png](/images/6/6-24.png)

**Giải thích hiển thị kế hoạch thực thi cho một câu lệnh truy vấn mà không chạy truy vấn**

- Lưu ý việc sử dụng các cột phân vùng trong câu lệnh SELECT và WHERE. Các cột đó nằm ở đâu trong định nghĩa bảng Spectrum của bạn?
- Lưu ý các bộ lọc được áp dụng ở mức phân vùng hoặc tệp trong tập dữ liệu Spectrum của truy vấn (so với tập dữ liệu Redshift Managed Storage).
- Nếu bạn thực sự chạy truy vấn (và không chỉ tạo kế hoạch giải thích), thời gian chạy có làm bạn ngạc nhiên không? Tại sao hoặc tại sao không?

```jsx
EXPLAIN
SELECT year, month, type, COUNT(*)
FROM adb305_view_NYTaxiRides
WHERE year = 2016 AND month IN (1,2) AND passenger_count = 4
GROUP BY 1,2,3 ORDER BY 1,2,3;
```

![image.png](/images/6/6-25.png)

![image.png](/images/6/6-26.png)

Lưu ý rằng **S3 Seq Scan** đã được thực hiện trên dữ liệu trên Amazon S3. Node **S3 Seq Scan** cho thấy bộ lọc: (passenger_count = 4) đã được xử lý trong lớp **Redshift Spectrum**.

Để cải thiện hiệu suất Redshift Spectrum, vui lòng tham khảo [https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-performance.html](https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-performance.html)


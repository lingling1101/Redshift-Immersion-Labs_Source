---
title : "Chia sẻ dữ liệu"
date :  "`r Sys.Date()`" 
weight : 4 
chapter : false
pre : " <b> 4. </b> "
---
**IV. Chia sẻ dữ liệu**

Bạn sẽ thực hiện các bước để chia sẻ dữ liệu giữa một endpoint serverless và một cluster được cung cấp.

**Nội dung**

- Trước khi bắt đầu
- Giới thiệu
- Xác định các không gian tên (namespaces)
- Tạo Data Share trên máy chủ sản xuất (producer)
- Truy vấn dữ liệu từ máy chủ tiêu thụ (consumer)
- Tạo schema ngoại vi trong máy chủ tiêu thụ
- Tải dữ liệu cục bộ và kết hợp với dữ liệu chia sẻ
- Trước khi rời khỏi

**4.1 Trước khi bắt đầu**

Hướng dẫn này giả định rằng bạn đã khởi chạy một Amazon Redshift Serverless endpoint và một cluster được cung cấp. Nếu bạn chưa làm như vậy, vui lòng xem phần [**Hướng dẫn bắt đầu**](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab1) và làm theo các hướng dẫn tại đó. Chúng tôi sẽ sử dụng [Amazon Redshift QueryEditorV2](https://console.aws.amazon.com/sqlworkbench/home) cho bài lab này.

**4.2 Giới thiệu**

*Một **datashare** là đơn vị chia sẻ dữ liệu trong Amazon Redshift. Sử dụng datashares để chia sẻ dữ liệu trong cùng một tài khoản AWS hoặc các tài khoản AWS khác. Đồng thời, chia sẻ dữ liệu để đọc từ các cluster Amazon Redshift khác nhau. Mỗi datashare liên kết với một cơ sở dữ liệu cụ thể trong cluster Amazon Redshift của bạn.*

*Các đối tượng datashare là các đối tượng từ các cơ sở dữ liệu cụ thể trên một cluster mà các quản trị viên của cluster sản xuất có thể thêm vào datashares để chia sẻ với các người tiêu thụ dữ liệu. Các đối tượng datashare chỉ đọc cho người tiêu thụ dữ liệu. Ví dụ về các đối tượng datashare bao gồm bảng, view và hàm do người dùng định nghĩa. Bạn có thể thêm các đối tượng datashare vào datashares trong khi tạo datashares hoặc chỉnh sửa một datashare vào bất kỳ lúc nào.*

**Tài liệu tham khảo:** [Chia sẻ dữ liệu trên AWS Redshift](https://aws.amazon.com/redshift/features/data-sharing/)

**4.3 Xác định không gian tên (namespace):**

Chúng ta sẽ chạy các câu lệnh SQL đơn giản để tìm không gian tên cho cả cụm (cluster) nhà sản xuất và cụm nhà tiêu dùng. Những giá trị này sẽ được sử dụng trong các hướng dẫn tiếp theo.

**Lưu ý:** Không gian tên là một định danh duy nhất toàn cầu (GUID) được tự động tạo ra trong quá trình tạo cụm Amazon Redshift và được gắn liền với cụm đó. Nó được sử dụng để tham chiếu duy nhất tới cụm Redshift.

Chạy lệnh sau trong trình chỉnh sửa truy vấn kết nối với endpoint serverless (producer - Serverless: workgroup-xxxxxxx). Ghi lại không gian tên từ kết quả đầu ra. Đảm bảo rằng trình chỉnh sửa kết nối với cluster sản xuất.

```jsx
-- This should be run on the producer 
select current_namespace;
```
![image.png](/images/4/4-1.png)

Chạy lệnh sau trong trình chỉnh sửa truy vấn kết nối với cluster được cung cấp (consumer - consumercluster-xxxxxxxxxx). Ghi lại không gian tên từ kết quả đầu ra. Đảm bảo rằng trình chỉnh sửa kết nối với cluster tiêu thụ.

```jsx
-- This should be run on consumer 
select current_namespace;
```

![image.png](/images/4/4-2.png)

![image.png](/images/4/4-3.png)

**4.4 Tạo data share trên máy chủ sản xuất (producer)**

Chạy các lệnh sau khi kết nối với endpoint serverless (producer - Serverless: workgroup-xxxxxxx) để tạo một data share và thêm bảng `customer` vào data share.

Thay thế `<<consumer namespace>>` bằng không gian tên của cluster tiêu thụ mà bạn đã ghi lại ở đầu bài lab này.

```jsx
-- Creating a datashare
CREATE DATASHARE cust_share SET PUBLICACCESSIBLE TRUE;

-- Adding schema to datashare
ALTER DATASHARE cust_share ADD SCHEMA public;

-- Adding customer tables to datshares.  We can add all the tables also if required
ALTER DATASHARE cust_share ADD TABLE public.customer;

-- View shared objects
show datashares;
select * from SVV_DATASHARE_OBJECTS;

-- Granting access to consumer cluster
Grant USAGE ON DATASHARE cust_share to NAMESPACE '<<consumer namespace>>'
```

![image.png](/images/4/4-4.png)

**4.5 Truy vấn dữ liệu từ máy chủ tiêu thụ (consumer)**

Vui lòng chạy các lệnh sau trên cluster được cung cấp (consumer - consumercluster-xxxxxxxxxx) để tạo một cơ sở dữ liệu cục bộ trên dữ liệu chia sẻ.

**LƯU Ý:** Thay thế `<<producer namespace>>` bằng không gian tên của cluster sản xuất mà bạn đã ghi lại ở đầu bài lab này.

```jsx
-- View shared objects
show datashares;
select * from SVV_DATASHARE_OBJECTS;

-- Create local database
CREATE DATABASE cust_db FROM DATASHARE cust_share OF NAMESPACE '<<producer namespace>';

-- Select query to check the count
select count(*) from cust_db.public.customer; -- count 15000000
```

![image.png](/images/4/4-5.png)



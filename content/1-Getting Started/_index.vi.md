---
title : "Bắt đầu"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 1. </b> "
---

### **1. Bắt đầu**

Chào mừng bạn đến với Redshift Immersion Day!!

Bạn sẽ có cơ hội trải nghiệm thực tế các tính năng quan trọng của Redshift trong ngày immersion này. Nó bao gồm các tính năng nền tảng như tải dữ liệu, chuyển đổi dữ liệu, chia sẻ dữ liệu, Redshift Spectrum, học máy và vận hành. Đối với các bài tập nâng cao, bạn nên sử dụng hội thảo Redshift Deep Dive.

Tất cả các bài tập đều được thiết kế để chạy trên tính năng serverless mới của Redshift. Amazon Redshift Serverless giúp việc chạy và mở rộng phân tích trở nên đơn giản mà không cần quản lý loại phiên bản, kích thước phiên bản, quản lý vòng đời, tạm dừng, tiếp tục, v.v. Nó tự động cung cấp và mở rộng năng lực tính toán của kho dữ liệu để mang lại hiệu suất nhanh cho ngay cả những khối lượng công việc đòi hỏi khắt khe và khó đoán nhất, và bạn chỉ phải trả tiền cho những gì bạn sử dụng. Chỉ cần tải dữ liệu của bạn và bắt đầu truy vấn ngay trong Amazon Redshift Query Editor hoặc công cụ phân tích kinh doanh (BI) yêu thích của bạn và tiếp tục tận hưởng hiệu suất giá tốt nhất cùng các tính năng SQL quen thuộc trong môi trường dễ sử dụng, không cần quản trị.

Hình dưới đây cho thấy các tính năng chính mà bạn sẽ làm việc và cách chúng được sử dụng trong môi trường kho dữ liệu điển hình.

![image.png](/images/1/1-111.png)

**Các tài nguyên AWS cần thiết để chạy các bài tập này bao gồm:**

- Một endpoint Amazon Redshift Serverless trong một môi trường mạng mới, bao gồm một VPC, 3 Subnet, một Internet Gateway và một Security Group để cho phép truy cập cục bộ vào cụm của bạn.
- Một cụm Redshift Provisioned trong cùng một môi trường mạng để sử dụng trong bài tập chia sẻ dữ liệu.
- Một vai trò mặc định được đính kèm vào các môi trường của bạn. Để tìm hiểu thêm về cách ủy quyền cho Amazon Redshift truy cập vào các dịch vụ AWS khác, xem tài liệu về [Tạo vai trò IAM mặc định cho Amazon Redshift](https://docs.aws.amazon.com/redshift/latest/mgmt/default-iam-role.html).

### **1.1 AWS-Sponsored Immersion Day**

Nếu bạn đang thực hiện các bài tập này như một phần của AWS-Sponsored Immersion Day, một tài khoản lab đã được tạo với các tài nguyên ở trên. Để biết hướng dẫn về cách kết nối với môi trường của bạn, xem [Tham gia sự kiện hội thảo](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab0).


### **1.2 Self-paced Immersion Day**
Nếu bạn đang tự thực hiện các bài tập này, hãy sử dụng mẫu CFN sau để khởi chạy các tài nguyên cần thiết trong tài khoản AWS của bạn.

[Launch Stack](https://console.aws.amazon.com/cloudformation/home?#/stacks/new?stackName=RedshiftImmersionLab&templateURL=https://s3-us-west-2.amazonaws.com/redshift-immersionday-labs/immersionserverless.yaml)


![image.png](/images/1/1-2.png)

![image.png](/images/1/1-31.png)

### **1.3 Cấu hình công cụ khách hàng - Query Editor V2**

Khi bạn đã có quyền truy cập vào tài khoản với các tài nguyên cần thiết, hãy kiểm tra xem bạn có thể kết nối với môi trường Redshift của mình hay không. Các bài tập này sử dụng Query Editor v2 dựa trên web của Redshift. Hãy điều hướng đến [Query Editor v2](https://ap-southeast-1.console.aws.amazon.com/sqlworkbench/home?region=ap-southeast-1#/client).

Bạn có thể cần phải cấu hình Query Editor.

![image.png](/images/1/1-4.png)

Ở phía bên trái, nhấp vào môi trường Redshift mà bạn muốn kết nối.

![image.png](/images/1/1-5.png)

Một cửa sổ pop-up sẽ hiện ra.

Nhập tên Database và user name. Nhấn "connect". Các thông tin đăng nhập này nên được sử dụng cho cả endpoint Serverless (workgroup-xxxxxxx) cũng như cụm đã cấp phát (consumercluster-xxxxxxxxxx).

```jsx
User Name: awsuser  
Password: Awsuser123
```
![image.png](/images/1/1-6.png)

### **1.4 Chạy truy vấn mẫu**

- Chạy truy vấn sau để liệt kê các người dùng trong cụm Redshift:

```jsx
select * from pg_user;
```

Nếu bạn nhận được kết quả sau, bạn đã thiết lập kết nối và phòng thí nghiệm này đã hoàn tất.

![image.png](/images/1/1-7.png)
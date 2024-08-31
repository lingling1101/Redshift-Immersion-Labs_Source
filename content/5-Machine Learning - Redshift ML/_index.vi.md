---
title : "Học máy - Redshift ML"
date :  "`r Sys.Date()`" 
weight : 5 
chapter : false
pre : " <b> 5. </b> "
---
**V. Học máy - Redshift ML**

Trong bài lab này, bạn sẽ tạo một mô hình sử dụng Redshift ML Auto.

**Nội dung**

- Trước khi bắt đầu
- Chuẩn bị dữ liệu
- Tạo mô hình
- Kiểm tra độ chính xác và chạy truy vấn suy diễn
- Giải thích
- Trước khi rời khỏi

**5.1 Trước khi bắt đầu**

Hướng dẫn này giả định rằng bạn đã khởi chạy một Amazon Redshift Serverless endpoint. Nếu bạn chưa làm như vậy, vui lòng xem phần [**Hướng dẫn bắt đầu**](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab1) và làm theo các hướng dẫn tại đó. Chúng tôi sẽ sử dụng [Amazon Redshift QueryEditorV2](https://console.aws.amazon.com/sqlworkbench/home) cho bài lab này.

**5.2 Chuẩn bị dữ liệu**

Tập dữ liệu Marketing Ngân hàng chứa các chiến dịch tiếp thị trực tiếp của một tổ chức ngân hàng Bồ Đào Nha. Các chiến dịch tiếp thị dựa trên các cuộc gọi điện thoại. Thường thì cần phải liên hệ nhiều lần với cùng một khách hàng để đánh giá xem sản phẩm (tiền gửi ngân hàng) có được đăng ký ('yes') hay không ('no').

Tập dữ liệu bao gồm các thuộc tính sau. Mục tiêu phân loại là dự đoán xem khách hàng có đăng ký vào tiền gửi kỳ hạn hay không.

**Các thuộc tính của dữ liệu khách hàng ngân hàng**

- age (số)
- jobtype
- marital
- education
- default
- housing
- loan
- contact
- month
- day_of_week
- duration
- campaign
- pdays
- previous
- poutcome
- emp.var.rate
- cons.price.idx
- cons.conf.idx
- euribor3m
- nr.employed

**Tài liệu tham khảo:** [Tập dữ liệu marketing ngân hàng](https://archive.ics.uci.edu/ml/datasets/bank+marketing)

Thực hiện các lệnh sau để tạo và tải bảng huấn luyện vào Redshift. Lưu ý trường bổ sung `y`, chứa giá trị 'yes' hoặc 'no' chỉ kết quả của việc đăng ký tiền gửi kỳ hạn. Dữ liệu huấn luyện sẽ được tải với dữ liệu lịch sử và được sử dụng để tạo mô hình.

```jsx
CREATE TABLE bank_details_training(
   age numeric,
   jobtype varchar,
   marital varchar,
   education varchar,
   "default" varchar,
   housing varchar,
   loan varchar,
   contact varchar,
   month varchar,
   day_of_week varchar,
   duration numeric,
   campaign numeric,
   pdays numeric,
   previous numeric,
   poutcome varchar,
   emp_var_rate numeric,
   cons_price_idx numeric,     
   cons_conf_idx numeric,     
   euribor3m numeric,
   nr_employed numeric,
   y boolean ) ;

COPY bank_details_training from 's3://redshift-downloads/redshift-ml/workshop/bank-marketing-data/training_data/' REGION 'us-east-1' IAM_ROLE default CSV IGNOREHEADER 1 delimiter ';';
```

![image.png](/images/5/5-1.png)

Thực hiện các câu lệnh sau để tạo và tải bảng suy luận trong Redshift. Dữ liệu này sẽ được sử dụng để mô phỏng dữ liệu mới mà chúng tôi có thể kiểm tra dựa trên mô hình.

```jsx
CREATE TABLE bank_details_inference(
   age numeric,
   jobtype varchar,
   marital varchar,
   education varchar,
   "default" varchar,
   housing varchar,
   loan varchar,
   contact varchar,
   month varchar,
   day_of_week varchar,
   duration numeric,
   campaign numeric,
   pdays numeric,
   previous numeric,
   poutcome varchar,
   emp_var_rate numeric,
   cons_price_idx numeric,     
   cons_conf_idx numeric,     
   euribor3m numeric,
   nr_employed numeric,
   y boolean ) ;

COPY bank_details_inference from 's3://redshift-downloads/redshift-ml/workshop/bank-marketing-data/inference_data/' REGION 'us-east-1' IAM_ROLE default CSV IGNOREHEADER 1 delimiter ';';
```

![image.png](/images/5/5-2.png)

**Tạo bucket S3**

Trước khi bạn tạo một mô hình, bạn cần tạo một bucket S3 để lưu trữ các kết quả trung gian. Truy cập vào S3 và tạo một bucket với tên `testanalytics` kết thúc bằng tên của bạn, một số ngẫu nhiên hoặc số tài khoản của bạn để có một tên duy nhất. Bạn có thể tìm số tài khoản của mình ở góc trên bên phải của bảng điều khiển quản lý. Mục đích là để tạo một bucket S3 với tên duy nhất.

![image.png](/images/5/5-3.png)

![image.png](/images/5/5-4.png)

**5.3 Tạo mô hình**

Hoàn thành Autopilot được tạo ra với ít đầu vào từ người dùng. Đây sẽ là một bài toán phân loại nhị phân, nhưng Autopilot sẽ chọn thuật toán phù hợp dựa trên dữ liệu và đầu vào.

Thực hiện lệnh sau để tạo mô hình. Thay thế `<<S3 bucket>>` bằng tên bucket mà bạn đã tạo ở trên.

```jsx

DROP MODEL model_bank_marketing;

CREATE MODEL model_bank_marketing
FROM (
SELECT    
   age ,
   jobtype ,
   marital ,
   education ,
   "default" ,
   housing ,
   loan ,
   contact ,
   month ,
   day_of_week ,
   duration ,
   campaign ,
   pdays ,
   previous ,
   poutcome ,
   emp_var_rate ,
   cons_price_idx ,     
   cons_conf_idx ,     
   euribor3m ,
   nr_employed ,
   y
FROM
    bank_details_training )
    TARGET y
FUNCTION func_model_bank_marketing
IAM_ROLE default
SETTINGS (
  S3_BUCKET '<<S3 bucket>>',
  MAX_RUNTIME 3600
  )
;
```

![image.png](/images/5/5-5.png)

Chạy lệnh sau để kiểm tra trạng thái của mô hình. Nó sẽ ở trạng thái TRAINING. Mô hình tạo sẽ mất ~ 60 phút để chạy.

```jsx
show model model_bank_marketing;
```

![image.png](/images/5/5-6.png)

**5.4 Kiểm tra độ chính xác và chạy truy vấn suy luận**

Hy vọng rằng bạn đã cho mô hình đủ thời gian (~60 phút) để hoàn tất quá trình đào tạo. Chạy cùng một câu lệnh SQL như trên để kiểm tra trạng thái của mô hình. Trạng thái của mô hình nên là 'Ready' để bạn có thể tiếp tục. Chú ý đến điểm số validation- nó sẽ nằm trong khoảng từ 0 đến 1, càng gần 1 thì mô hình càng tốt.

```jsx
show model model_bank_marketing;
```

![image.png](/images/5/5-7.png)

Kiểm tra độ chính xác và chạy truy vấn suy luận. Chạy các truy vấn sau - truy vấn đầu tiên kiểm tra độ chính xác của mô hình và truy vấn thứ hai sẽ sử dụng hàm được tạo bởi mô hình đã được xây dựng để suy diễn và so với tập dữ liệu trong bảng suy diễn

```
bank_details_inference
```

```jsx
--Inference/Accuracy on inference data

WITH infer_data
 AS (
    SELECT  y as actual, func_model_bank_marketing(age,jobtype,marital,education,"default",housing,loan,contact,month,day_of_week,duration,campaign,pdays,previous,poutcome,emp_var_rate,cons_price_idx,cons_conf_idx,euribor3m,nr_employed) AS predicted,
     CASE WHEN actual = predicted THEN 1::INT
         ELSE 0::INT END AS correct
    FROM bank_details_inference
    ),
 aggr_data AS (
     SELECT SUM(correct) as num_correct, COUNT(*) as total FROM infer_data
 )
 SELECT (num_correct::float/total::float) AS accuracy FROM aggr_data;

--Predict how many will subscribe for term deposit vs not subscribe

WITH term_data AS ( SELECT func_model_bank_marketing( age,jobtype,marital,education,"default",housing,loan,contact,month,day_of_week,duration,campaign,pdays,previous,poutcome,emp_var_rate,cons_price_idx,cons_conf_idx,euribor3m,nr_employed) AS predicted
FROM bank_details_inference )
SELECT
CASE WHEN predicted = 'Y'  THEN 'Yes-will-do-a-term-deposit'
     WHEN predicted = 'N'  THEN 'No-term-deposit'
     ELSE 'Neither' END as deposit_prediction,
COUNT(1) AS count
from term_data GROUP BY 1;.
```

![image.png](/images/5/5-8.png)

![image.png](/images/5/5-9.png)

**5.5 Giải thích mô hình**

Bạn có thể xác định các thuộc tính nào đang đóng góp tích cực cho dự đoán và mức độ đóng góp của chúng bằng cách tạo báo cáo giải thích mô hình. Vui lòng chạy lệnh sau để xem giải thích mô hình:

```jsx
SELECT explain_model('model_bank_marketing');
```

![image.png](/images/5/5-10.png)

Bạn sẽ nhận được thông báo cho biết mô hình chưa được đào tạo đủ lâu để tạo báo cáo giải thích. Điều này là bình thường vì mô hình được đào tạo với `MAX_RUNTIME` là 3600 giây. Thông thường, bạn nên tăng `MAX_RUNTIME` lên 9600 giây hoặc hơn để Redshift có thể tạo báo cáo giải thích. Điều này cung cấp đủ thời gian để hoàn thành các bước báo cáo giải thích của mô hình.


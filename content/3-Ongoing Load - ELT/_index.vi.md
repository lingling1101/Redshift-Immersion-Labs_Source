---
title : "Tải dữ liệu liên tục - ELT"
date :  "`r Sys.Date()`" 
weight : 3 
chapter : false
pre : " <b> 3. </b> "
---

**3. Quá trình tải liên tục - ELT**

**Nội dung**

- Trước khi bắt đầu
- Stored Procedures - Tải dữ liệu đang diễn ra
- Stored Procedures - Xử lý ngoại lệ
- Materialized Views
- User Defined Functions
- Trước khi kết thúc

**3.1 Trước khi bắt đầu**

Bản lab này giả định rằng bạn đã khởi chạy một điểm cuối Amazon Redshift Serverless. Nếu bạn chưa làm như vậy, vui lòng xem phần Bắt đầu và làm theo các hướng dẫn tại đó. Chúng ta sẽ sử dụng Amazon Redshift QueryEditorV2 cho bản lab này.

Với ETL, việc chuyển đổi dữ liệu diễn ra ở một máy chủ trung gian ETL trước khi nạp nó vào Data Warehouse. Sơ đồ sau minh họa quy trình làm việc của ETL:

![image.png](/images/3/3-1.png)

Với ELT (Extract, Load, Transform), việc chuyển đổi dữ liệu diễn ra ngay trong kho dữ liệu mục tiêu thay vì cần một máy chủ trung gian ETL. Cách tiếp cận này tận dụng khả năng của các công cụ cơ sở dữ liệu Redshift hỗ trợ xử lý song song quy mô lớn (MPP). Sơ đồ sau minh họa quy trình làm việc của ELT:

![image.png](/images/3/3-2.png)

**3.2 Stored Procedures - Tải dữ liệu đang diễn ra**

Stored procedures thường được sử dụng để đóng gói logic cho việc chuyển đổi dữ liệu, xác thực dữ liệu, và logic đặc thù của doanh nghiệp. Bằng cách kết hợp nhiều bước SQL vào một stored procedure, bạn có thể giảm bớt số lần trao đổi giữa ứng dụng của bạn và cơ sở dữ liệu. Một stored procedure có thể bao gồm ngôn ngữ định nghĩa dữ liệu (DDL) và ngôn ngữ thao tác dữ liệu (DML) bên cạnh các truy vấn SELECT. Stored procedure không bắt buộc phải trả về giá trị. Bạn có thể sử dụng ngôn ngữ thủ tục PL/pgSQL, bao gồm các vòng lặp và biểu thức điều kiện, để điều khiển luồng logic.

Hãy xem cách bạn có thể tạo và gọi stored procedure trong Redshift. Mục tiêu ở đây là cập nhật gia tăng dữ liệu lineitem. Hãy thực thi truy vấn sau để tạo bảng lineitem staging:

```jsx
create table stage_lineitem (
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
  L_COMMENT varchar(44));
```
![image.png](/images/3/3-33.png)

Thực thi script dưới đây để tạo một stored procedure. Stored procedure này thực hiện các nhiệm vụ sau:

1. Xóa sạch bảng staging để loại bỏ dữ liệu cũ.
2. Tải dữ liệu vào bảng `stage_lineitem` bằng lệnh `COPY`.
3. Hợp nhất các bản ghi được cập nhật vào bảng `lineitem` hiện có. Hàm `MERGE` sẽ cập nhật bảng đích dựa trên dữ liệu mới trong bảng staging.

```jsx
CREATE OR REPLACE PROCEDURE lineitem_incremental()
AS $$
BEGIN

truncate stage_lineitem;  

copy stage_lineitem from 's3://redshift-immersionday-labs/data/lineitem/lineitem.tbl.340.lzo'
iam_role default
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy stage_lineitem from 's3://redshift-immersionday-labs/data/lineitem/lineitem.tbl.341.lzo'
iam_role default
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy stage_lineitem from 's3://redshift-immersionday-labs/data/lineitem/lineitem.tbl.342.lzo'
iam_role default
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

merge into lineitem
using stage_lineitem
on stage_lineitem.l_orderkey = lineitem.l_orderkey
and stage_lineitem.l_linenumber = lineitem.l_linenumber
remove duplicates
;

END;
$$ LANGUAGE plpgsql;
```

![image.png](/images/3/3-44.png)

Trước khi bạn gọi stored procedure này, hãy thu thập tổng số hàng từ bảng `lineitem`.

```jsx
SELECT count(*) FROM "dev"."public"."lineitem"; --303008217
```

![image.png](/images/3/3-5.png)

Gọi stored procedure này bằng cách sử dụng câu lệnh `CALL`. Khi thực thi, nó sẽ thực hiện một quá trình tải gia tăng:

```jsx
call lineitem_incremental();
```

![image.png](/images/3/3-6.png)

Sau khi bạn gọi stored procedure này, hãy xác minh xem tổng số hàng trong bảng `lineitem` đã thay đổi hay chưa.

```jsx
SELECT count(*) FROM "dev"."public"."lineitem"; --306008217
```

![image.png](/images/3/3-7.png)

**3.3 Stored Procedures - Xử lý Ngoại lệ**

Phần tiếp theo đề cập đến việc xử lý ngoại lệ trong stored procedures. Stored procedures hỗ trợ xử lý ngoại lệ theo định dạng sau:

```jsx
Example pseudocode:

BEGIN
	statements
EXCEPTION
	WHEN OTHERS THEN
		statements
END;
```

Các ngoại lệ được xử lý trong stored procedures khác nhau dựa trên chế độ nguyên tử (atomic) hoặc không nguyên tử (non-atomic) được chọn.

- **Nguyên tử (atomic) (mặc định):** Các ngoại lệ (lỗi) luôn được đưa ra lại.
- **Không nguyên tử (non-atomic):** Các ngoại lệ được xử lý và bạn có thể chọn việc đưa ra lại hay không.

**Lưu ý:** Khi gọi một stored procedure từ một stored procedure khác, các stored procedures phải có chế độ giao dịch (transaction mode) giống nhau, nếu không bạn sẽ thấy lỗi sau: "Stored procedure created in one transaction mode cannot be invoked from another procedure in a different transaction mode."

Để bắt đầu, chúng ta cần tạo một bảng cho các stored procedures trong phần này.

```jsx
CREATE TABLE stage_lineitem2 (LIKE stage_lineitem);
```

![image.png](/images/3/3-88.png)

Tiếp theo, chúng ta sẽ tạo một bảng để ghi lại các thông báo lỗi.

```jsx
CREATE TABLE procedure_log
(log_timestamp timestamp, procedure_name varchar(100), error_message varchar(255));
```

![image.png](/images/3/3-99.png)

**3.4 Nguyên tử (Atomic)**

Các stored procedures trong Redshift mặc định là nguyên tử, điều này có nghĩa là bạn có quyền kiểm soát giao dịch cho các câu lệnh trong stored procedure. Nếu bạn chọn không bao gồm các lệnh

```
COMMIT
```

hoặc

```
ROLLBACK
```

trong stored procedure, thì tất cả các câu lệnh sẽ được xử lý trong một giao dịch duy nhất. Quan trọng là trong chế độ nguyên tử mặc định, các ngoại lệ luôn được đưa ra lại.

```jsx
CREATE OR REPLACE PROCEDURE pr_divide_by_zero() AS
$$
DECLARE
	v_int int;
BEGIN
	v_int := 1/0;
EXCEPTION
	WHEN OTHERS THEN
		INSERT INTO procedure_log VALUES (getdate(), 'pr_divide_by_zero', sqlerrm);
		RAISE INFO 'pr_divide_by_zero: %', sqlerrm;
END;
$$
LANGUAGE plpgsql;
```

![image.png](/images/3/3-010.png)

Stored procedure trên được thiết kế để gặp lỗi chia cho số 0. Xử lý ngoại lệ trong stored procedure này lưu trữ thông báo lỗi vào bảng `procedure_log` và sau đó thực hiện lệnh `RAISE INFO` để thông báo cho procedure gọi biết về lỗi. Vì đây là một procedure nguyên tử (atomic), điều này sẽ dẫn đến việc lỗi được đưa ra lại ngay cả khi khối ngoại lệ không đưa ra lỗi. Mặc dù nó chỉ đưa ra thông báo thông tin (INFO), Redshift sẽ ghi đè điều này ở chế độ mặc định và một lỗi (ERROR) sẽ được đưa ra.

```jsx
CREATE OR REPLACE PROCEDURE pr_insert_stage() AS
$$
BEGIN
	TRUNCATE stage_lineitem2;

	INSERT INTO stage_lineitem2
	SELECT * FROM stage_lineitem;

	call pr_divide_by_zero();
EXCEPTION
	WHEN OTHERS THEN
		RAISE EXCEPTION 'pr_insert_stage: %', sqlerrm;
END;
$$
LANGUAGE plpgsql;
```

![image.png](/images/3/3-011.png)

Quy trình được lưu trữ thứ hai này sẽ chèn dữ liệu vào bảng stage_lineitem2 rồi gọi quy trình sẽ bị lỗi theo thiết kế.

```jsx
CALL pr_insert_stage();
```

![image.png](/images/3/3-012.png)

Bạn sẽ thấy hai thông báo:

```jsx
INFO:  pr_divide_by_zero: division by zero
ERROR:  pr_insert_stage: division by zero
```

Lưu ý rằng thông báo INFO là từ `pr_divide_by_zero`, trong khi lỗi (ERROR) là từ `pr_insert_stage`. Lỗi đã được tự động đưa ra lại, vì vậy `pr_insert_stage()` đã đưa ra lỗi (ERROR).

Bây giờ, hãy kiểm tra số lượng hàng trong bảng `stage_lineitem2`:

```jsx
SELECT COUNT(*) FROM stage_lineitem2;
```

![image.png](/images/3/3-13.png)

Vì stored procedure là nguyên tử (atomic), dữ liệu được chèn vào bảng stage đã bị hoàn tác (rolled back) do lỗi xảy ra sau đó, nên số lượng hàng trong bảng `stage_lineitem2` là bằng không.

```jsx
SELECT * FROM procedure_log ORDER BY log_timestamp DESC;
```

![image.png](/images/3/3-014.png)

Bạn sẽ thấy rằng lỗi thủ tục pr_divide_by_zero đã được ghi lại trong khối ngoại lệ.

**3.5 Không nguyên tử (Non-atomic)**

Stored procedures cũng có thể được tạo với tùy chọn `NONATOMIC`, điều này sẽ tự động `COMMIT` sau mỗi câu lệnh. Khi xảy ra ngoại lệ lỗi (ERROR) trong một stored procedure, ngoại lệ không phải lúc nào cũng được đưa ra lại. Thay vào đó, bạn có thể chọn "xử lý" lỗi và tiếp tục các câu lệnh tiếp theo trong stored procedure.

Các stored procedures sau đây giống hệt với các phiên bản nguyên tử, ngoại trừ việc mỗi stored procedure bây giờ bao gồm tùy chọn `NONATOMIC`. Chế độ này tự động cam kết (commit) các câu lệnh bên trong procedure và không tự động đưa ra lại các lỗi.

```jsx
CREATE OR REPLACE PROCEDURE pr_divide_by_zero_v2() NONATOMIC AS
$$
DECLARE
	v_int int;
BEGIN
	v_int := 1/0;
EXCEPTION
	WHEN OTHERS THEN
		INSERT INTO procedure_log VALUES (getdate(), 'pr_divide_by_zero_v2', sqlerrm);
		RAISE INFO 'pr_divide_by_zero_v2: %', sqlerrm;
END;
$$
LANGUAGE plpgsql;

CREATE OR REPLACE PROCEDURE pr_insert_stage_v2() NONATOMIC AS
$$
BEGIN
	TRUNCATE stage_lineitem2;

	INSERT INTO stage_lineitem2
	SELECT * FROM stage_lineitem;

	call pr_divide_by_zero_v2();
EXCEPTION
	WHEN OTHERS THEN
		RAISE EXCEPTION 'pr_insert_stage_v2: %', sqlerrm;
END;
$$
LANGUAGE plpgsql;
```

![image.png](/images/3/3-015.png)

```jsx
CALL pr_insert_stage_v2();
```

![image.png](/images/3/3-16.png)

Bạn sẽ nhận được thông báo:

```jsx
INFO:  pr_divide_by_zero: division by zero
```

Hãy nhớ rằng trong chế độ nguyên tử mặc định, bạn đã thấy hai thông báo, trong khi ở chế độ này, lỗi (ERROR) đã được xử lý và ngoại lệ không được đưa ra lại.

Bây giờ, hãy kiểm tra số lượng hàng trong bảng `stage_lineitem2`:

```jsx
SELECT COUNT(*) FROM stage_lineitem2;
```

![image.png](/images/3/3-017.png)

Vì stored procedure là không nguyên tử (non-atomic), dữ liệu được chèn vào bảng không bị hoàn tác. Câu lệnh chèn đã được tự động cam kết. Số lượng hàng trong bảng `stage_lineitem2` là 4,155,141.

```jsx
SELECT * FROM procedure_log ORDER BY log_timestamp DESC;
```

![image.png](/images/3/3-018.png)

Bạn sẽ thấy rằng lỗi thủ tục pr_divide_by_zero_v2 đã được ghi lại trong khối ngoại lệ giống như trong phiên bản nguyên tử.

**3.6 Materialized Views**

Trong môi trường kho dữ liệu, các ứng dụng thường cần thực hiện các truy vấn phức tạp trên các bảng lớn—ví dụ, các câu lệnh SELECT thực hiện các phép nối nhiều bảng và tổng hợp dữ liệu trên các bảng chứa hàng tỷ hàng. Việc xử lý các truy vấn này có thể tốn kém về mặt tài nguyên hệ thống và thời gian để tính toán kết quả.

Materialized views trong Amazon Redshift cung cấp một cách để giải quyết các vấn đề này. Materialized view chứa một tập hợp kết quả đã được tính toán trước, dựa trên truy vấn SQL trên một hoặc nhiều bảng cơ sở. Tại đây, bạn sẽ học cách tạo, truy vấn và làm mới (refresh) một materialized view.

Hãy lấy một ví dụ khi bạn muốn tạo một báo cáo về các nhà cung cấp hàng đầu dựa trên số lượng hàng đã giao. Truy vấn này sẽ kết hợp các bảng lớn như `lineitem` và `suppliers` và quét một lượng dữ liệu lớn. Bạn có thể viết một truy vấn như sau:

```jsx
select n_name, s_name, l_shipmode,
  SUM(L_QUANTITY) Total_Qty
from lineitem
join supplier on l_suppkey = s_suppkey
join nation on s_nationkey = n_nationkey
where datepart(year, L_SHIPDATE) > 1997
group by 1,2,3
order by 3 desc
limit 1000;
```

![image.png](/images/3/3-19.png)

Truy vấn này mất thời gian để thực thi và vì nó đang quét một lượng lớn dữ liệu nên sẽ sử dụng nhiều tài nguyên I/O và CPU. Hãy tưởng tượng một tình huống mà nhiều người dùng trong tổ chức cần lấy các chỉ số ở mức nhà cung cấp như trên. Mỗi người có thể viết các truy vấn nặng tương tự, điều này có thể tốn thời gian và tốn kém. Thay vì làm như vậy, bạn có thể sử dụng một materialized view để lưu trữ các kết quả đã được tính toán trước, giúp tăng tốc cho các truy vấn có thể dự đoán và lặp lại.

Amazon Redshift cung cấp một số phương pháp để giữ cho các materialized views luôn được cập nhật. Bạn có thể cấu hình tùy chọn tự động làm mới (auto refresh) để làm mới các materialized views khi các bảng cơ sở được cập nhật. Quá trình tự động làm mới sẽ chạy vào thời điểm khi tài nguyên của cụm sẵn có để giảm thiểu sự gián đoạn cho các tác vụ khác.

Thực hiện truy vấn dưới đây để tạo một materialized view, trong đó tổng hợp dữ liệu từ bảng `lineitem` theo mức nhà cung cấp. Lưu ý rằng tùy chọn `AUTO REFRESH` được đặt thành `YES` và chúng tôi đã bao gồm các cột bổ sung trong materialized view để các người dùng khác có thể tận dụng dữ liệu đã được tổng hợp này.

```jsx
CREATE MATERIALIZED VIEW supplier_shipmode_agg
AUTO REFRESH YES AS
select l_suppkey, l_shipmode, datepart(year, L_SHIPDATE) l_shipyear,
  SUM(L_QUANTITY)	TOTAL_QTY,
  SUM(L_DISCOUNT) TOTAL_DISCOUNT,
  SUM(L_TAX) TOTAL_TAX,
  SUM(L_EXTENDEDPRICE) TOTAL_EXTENDEDPRICE  
from LINEITEM
group by 1,2,3;
```

![image.png](/images/3/3-0020.png)

Bây giờ hãy thực hiện truy vấn bên dưới đã được viết lại để sử dụng chế độ xem cụ thể hóa. Lưu ý sự khác biệt về thời gian thực hiện truy vấn. Bạn nhận được kết quả tương tự trong vài giây.

```jsx
select n_name, s_name, l_shipmode,
  SUM(TOTAL_QTY) Total_Qty
from supplier_shipmode_agg
join supplier on l_suppkey = s_suppkey
join nation on s_nationkey = n_nationkey
where l_shipyear > 1997
group by 1,2,3
order by 3 desc
limit 1000;
```

![image.png](/images/3/3-21.png)

Một tính năng mạnh mẽ khác của materialized view là khả năng tự động viết lại truy vấn (auto query rewrite). Amazon Redshift có thể tự động viết lại các truy vấn để sử dụng materialized view, ngay cả khi truy vấn không tham chiếu rõ ràng đến materialized view.

Bây giờ, hãy chạy lại truy vấn gốc của bạn, truy vấn này tham chiếu đến bảng `lineitem`, và xem truy vấn này thực thi nhanh hơn vì Redshift đã viết lại truy vấn để tận dụng materialized view thay vì bảng cơ sở.

```jsx
select n_name, s_name, l_shipmode, SUM(L_QUANTITY) Total_Qty
from lineitem
join supplier on l_suppkey = s_suppkey
join nation on s_nationkey = n_nationkey
where datepart(year, L_SHIPDATE) > 1997
group by 1,2,3
order by 3 desc
limit 1000;
```

![image.png](/images/3/3-22.png)

Bạn có thể xác minh rằng người viết lại truy vấn đang sử dụng MV bằng cách chạy thao tác giải thích:

```jsx
explain
select n_name, s_name, l_shipmode, SUM(L_QUANTITY) Total_Qty
from lineitem
join supplier on l_suppkey = s_suppkey
join nation on s_nationkey = n_nationkey
where datepart(year, L_SHIPDATE) > 1997
group by 1,2,3
order by 3 desc
limit 1000;
```

![image.png](/images/3/3-23.png)

![image.png](/images/3/3-232.png)

> Viết các truy vấn bổ sung có thể tận dụng chế độ xem được thực thể hóa của bạn nhưng không tham chiếu trực tiếp đến nó. Ví dụ: Tổng giá mở rộng theo khu vực.

**3.7 Tổng hợp**

Hãy xem liệu Redshift có tự động làm mới materialized view sau khi dữ liệu bảng `lineitem` thay đổi không.

Vui lòng thu thập một chỉ số sử dụng materialized view. Chúng ta sẽ so sánh giá trị này sau khi dữ liệu của bảng cơ sở thay đổi.

```jsx
select SUM(TOTAL_QTY) Total_Qty from supplier_shipmode_agg;
```

![image.png](/images/3/3-24.png)

Hãy xóa một số dữ liệu đã tải trước đó khỏi bảng  *lineitem*.

```jsx
delete from lineitem
using orders
where l_orderkey = o_orderkey
and datepart(year, o_orderdate) = 1998 and datepart(month, o_orderdate) = 8;
```

![image.png](/images/3/3-25.png)

Chạy các truy vấn dưới đây trên materialized view và so sánh với giá trị bạn đã ghi lại trước đó. Bạn sẽ thấy giá trị tổng (SUM) đã thay đổi, điều này cho thấy Redshift đã nhận diện các thay đổi đã xảy ra trong bảng cơ sở hoặc các bảng cơ sở, và sau đó áp dụng các thay đổi đó vào materialized view.

**Lưu ý:** Việc làm mới materialized view là bất đồng bộ (asynchronous). Trong bài lab này, hãy dự đoán khoảng 5 phút để dữ liệu được làm mới sau khi bạn gọi stored procedure `lineitem_incremental`.

```jsx
select SUM(TOTAL_QTY) Total_Qty from supplier_shipmode_agg;
```

![image.png](/images/3/3-26.png)

**3.8 Hàm người dùng định nghĩa (User Defined Functions)**

Redshift hỗ trợ hàm người dùng định nghĩa (UDF) kiểu scalar bằng cách sử dụng câu lệnh SQL SELECT hoặc chương trình Python. Ví dụ dưới đây tạo một hàm Python so sánh hai số và trả về giá trị lớn hơn:

```jsx
create function f_py_greater (a float, b float)
  returns float
stable
as $$
  if a > b:
    return a
  return b
$$ language plpythonu;

select f_py_greater (l_extendedprice, l_discount) from lineitem limit 10
```


![image.png](/images/3/3-272.png)
![image.png](/images/3/3-27.png)
Ví dụ sau đây tạo hàm SQL so sánh hai số và trả về giá trị lớn hơn:

```jsx
create function f_sql_greater (float, float)
  returns float
stable
as $$
  select case when $1 > $2 then $1
    else $2
  end
$$ language sql;  

select f_sql_greater (l_extendedprice, l_discount) from lineitem limit 10
```

![image.png](/images/3/3-28.png)



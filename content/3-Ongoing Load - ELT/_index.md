---
title : "Ongoing Load - ELT"
date : "`r Sys.Date()`"
weight : 3
chapter : false
pre : " <b> 3. </b> "
---

### **3. Ongoing Load - ELT**

### **Contents**
- Before you begin
- Stored procedures - Ongoing loads
- Stored procedures - Exception handling
- Materialized views
- User defined functions
- Before you leave

### **3.1 Before you begin**

This lab assumes that you have launched an Amazon Redshift Serverless endpoint. If you have not already done so, please see Getting Started and follow the instructions there. We will use Amazon Redshift QueryEditorV2  for this lab.

With ETL, data transformation happens in a middle-tier ETL server before loading it on the Data Warehouse. The following diagram illustrates the ETL workflow:

![image.png](/images/3/3-1.png)

With ELT, data transformation happens in the target data warehouse rather than requiring a middle-tier ETL server. This approach takes advantage of Redshift database engines that support massively parallel processing (MPP). The following diagram illustrates the ELT workflow:

![image.png](/images/3/3-2.png)

### **3.2 Stored Procedures - Ongoing loads**

Stored procedures are commonly used to encapsulate logic for data transformation, data validation, and business-specific logic. By combining multiple SQL steps into a stored procedure, you can reduce round trips between your applications and the database. A stored procedure can incorporate data definition language (DDL) and data manipulation language (DML) in addition to SELECT queries. A stored procedure doesn’t have to return a value. You can use the PL/pgSQL procedural language, including looping and conditional expressions, to control logical flow.

Let’s see how you can create and invoke stored procedure in Redshift. Here our goal is to incrementally refresh the lineitem data. Execute the following query to create lineitem staging table:

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

Execute below script to create a stored procedure. This stored procedure performs following tasks:

1. Truncate staging table to clean up old data
2. Load data in the `stage_lineitem` table using the `COPY` command.
3. Merge updated records in existing `lineitem` table. The `MERGE`  function will update the target table based on the new data in the staging table.

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

Before you call this stored procedure, capture total #rows from lineitem table

```jsx
SELECT count(*) FROM "dev"."public"."lineitem"; --303008217
```

![image.png](/images/3/3-5.png)

Call this stored procedure using CALL statement. When executed it will perform an incremental load:

```jsx
call lineitem_incremental();
```

![image.png](/images/3/3-6.png)

After you call this stored procedure, verify total #rows from lineitem table has changed


```jsx
SELECT count(*) FROM "dev"."public"."lineitem"; --306008217
```

![image.png](/images/3/3-7.png)

### **3.3 Stored Procedures - Exception Handling**

This next section covers exception handling within stored procedures. Stored procedures support exception handling in the following format:

```jsx
Example pseudocode:

BEGIN
	statements
EXCEPTION
	WHEN OTHERS THEN
		statements
END;
```

Exceptions are handled in stored procedures differently based on the atomic or non-atomic mode chosen.

- **Atomic (default):** exceptions (errors) are always re-raised
- **Non-atomic:** exceptions are handled and you can choose to re-raise or not

> **Note:** When calling a stored procedure from another, the stored procedures must have matching transaction modes or else you will see the following error: Stored procedure created in one transaction mode cannot be invoked from another procedure in a different transaction mode

To get started, we need to first create a table for the stored procedures in this section.

```jsx
CREATE TABLE stage_lineitem2 (LIKE stage_lineitem);
```

![image.png](/images/3/3-88.png)

Next, we will create a table that will capture the error messages.


```jsx
CREATE TABLE procedure_log
(log_timestamp timestamp, procedure_name varchar(100), error_message varchar(255));
```

![image.png](/images/3/3-99.png)

### **3.4 Atomic**

Stored procedures in Redshift are atomic by default which means you have transaction control for the statements in the procedure. If you choose to not include COMMIT or ROLLBACK in the stored procedure, then all statements are handled in a single transaction. It is important to know that in the default atomic mode, exceptions are always re-raised.

```
COMMIT
```

or

```
ROLLBACK
```

in the stored procedure, then all statements are handled in a single transaction. It is important to know that in the default atomic mode, exceptions are always re-raised.

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

The above stored procedure is designed to fail with a division by zero error. The exception handling in this stored procedure stores the error message in the procedure_log table and then executes a RAISE INFO so the calling procedure is aware of the error. Because this is an atomic procedure, this will cause a re-raise of the error even though the exception block doesn't raise an error. It raises an INFO but Redshift overrides this in the default mode and an ERROR is raised.


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

This second stored procedure inserts data into the stage_lineitem2 table and then calls the procedure that will fail by design.


```jsx
CALL pr_insert_stage();
```

![image.png](/images/3/3-012.png)

You should see two messages:

```jsx
INFO:  pr_divide_by_zero: division by zero
ERROR:  pr_insert_stage: division by zero
```

Notice that the INFO message is from `pr_divide_by_zero` while the ERROR is from `pr_insert_stage`. The error was automatically re-raised so `pr_insert_stage(` raises an ERROR.

Now, check the number of rows in `stage_lineitem2`;

```jsx
SELECT COUNT(*) FROM stage_lineitem2;
```

![image.png](/images/3/3-13.png)

Because the stored procedure is atomic, the data inserted into the stage table is rolled back because of the subsequent error so the number of rows in `stage_lineitem2` is zero.

```jsx
SELECT * FROM procedure_log ORDER BY log_timestamp DESC;
```

![image.png](/images/3/3-014.png)

You should see that the pr_divide_by_zero procedure error was captured in the exception block.

### **3.5 Non-atomic**

Stored procedures can also be created with the `NONATOMIC` option which will automatically `COMMIT` after each statement. When an ERROR exception happens in a stored procedure, the exception is not always re-raised. Instead, you can choose to "handle" the error and continue subsequent statements in the stored procedure.

The following stored procedures are identical to the atomic versions except each now includes the `NONATOMIC` option. This mode automatically commits the statements inside the procedure and doesn't automatically re-raise errors.

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

You should see one message:

```jsx
INFO:  pr_divide_by_zero: division by zero
```

Remember that in the default atomic mode, you saw two messages where in this mode, the ERROR was handled and the exception wasn't re-raised.

Now, check the number of rows in `stage_lineitem2`;

```jsx
SELECT COUNT(*) FROM stage_lineitem2;
```

![image.png](/images/3/3-017.png)

Because the stored procedure is non-atomic, the data inserted into the table are not rolled back. The insert statement was automatically committed. The number of rows in `stage_lineitem2` is 4155141.

```jsx
SELECT * FROM procedure_log ORDER BY log_timestamp DESC;
```

![image.png](/images/3/3-018.png)

You should see that the pr_divide_by_zero_v2 procedure error was captured in the exception block just like in the atomic version.

### **3.6 Materialized Views**

In a data warehouse environment, applications often need to perform complex queries on large tables—for example, SELECT statements that perform multi-table joins and aggregations on the tables that contain billions of rows. Processing these queries can be expensive in terms of system resources and the time it takes to compute the results.

Materialized views in Amazon Redshift provide a way to address these issues. A materialized view contains a precomputed result set, based on SQL query over one or more base tables. Here you will learn how to create, query and refresh a materialized view.

Let’s take an example where you want to generate a report of the top suppliers by shipped quantity. This will join large tables like and `lineitem`, and `suppliers` and scan a large quantity of data. You might write a query like the following:

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

This query takes time to execute and because it is scanning a large amount of data will use a lot of I/O & CPU resources. Think of a situation, where multiple users in the organization need get supplier-level metrics like the above. Each may write similarly heavy queries which can be time consuming and expensive operations. Instead of that you can use a materialized view to store precomputed results for speeding up queries that are predictable and repeated.

Amazon Redshift provides a few methods to keep materialized views up-to-date. You can configure the automatic refresh option to refresh materialized views when base tables of mare updated. The auto refresh operation runs at a time when cluster resources are available to minimize disruptions to other workloads.

Execute below query to create materialized view which aggregates the lineitem data to the supplier level. Note, the `AUTO REFRESH` option is set to YES and we've included additional columns in our MV in case other users can take advantage of this aggregated data.

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

Now execute the below query which has been re-written to use the materialized view. Note the difference in query execution time. You get the same results in few seconds.

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

Another powerful feature of Materialized view is auto query rewrite. Amazon Redshift can automatically rewrite queries to use materialized views, even when the query doesn't explicitly reference a materialized view.

Now, re-run your original query which references the lineitem table and see this query now executes faster because Redshift has re-written this query to leverage the materialized view instead of base table.

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

You can verify that the query re-writer is using the MV by running an explain operation:

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

> Write additional queries which can leverage your materialized view but which do not directly reference it. For example, Total Extendedprice by Region.

### **3.7 Bringing it together**

Let’s see if Redshift is automatically refresh materialized view after lineitem table Data Changes.

Please capture a metric using the materialized view. We'll compare this value after base table data changes.

```jsx
select SUM(TOTAL_QTY) Total_Qty from supplier_shipmode_agg;
```

![image.png](/images/3/3-24.png)

let's delete some previously loaded data from the *lineitem* table.

```jsx
delete from lineitem
using orders
where l_orderkey = o_orderkey
and datepart(year, o_orderdate) = 1998 and datepart(month, o_orderdate) = 8;
```

![image.png](/images/3/3-25.png)

Run the below queries on the MV and compare with the value you had noted previously. You will see SUM has changed which indicates that Redshift has identified changes that have taken place in the base table or tables, and then applied those changes to the materialized view.

> **Notion:** the materialized view refresh is asynchronous. For this lab, expect ~5min for the data to be refreshed after you called the `lineitem_incremental` procedure:

```jsx
select SUM(TOTAL_QTY) Total_Qty from supplier_shipmode_agg;
```

![image.png](/images/3/3-26.png)

### **3.8 User Defined Functions**

Redshift supports scalar user-defined function (UDF) using either a SQL SELECT clause or a Python program. The following example creates a Python function that compares two numbers and returns the larger value:

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

The following example creates a SQL function that compares two numbers and returns the larger value:

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

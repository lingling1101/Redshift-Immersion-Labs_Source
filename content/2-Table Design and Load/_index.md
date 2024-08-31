---
title : "Table Design and Load"
date : "`r Sys.Date()`"
weight : 2
chapter : false
pre : " <b> 2. </b> "
---
**II. Table Design and Load**

**Contents:**

- Before you begin
- Create tables
- Loading data
-Troubleshooting loads
- Automatic table maintenance - ANALYZE and VACUUM
- Before you leave

**2.1 Before you begin**

This lab assumes that you have launched an Amazon Redshift Serverless endpoint. If you have not already done so, please see [Getting Started](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab1) and follow the instructions there. We will use Amazon Redshift [QueryEditorV2](https://ap-southeast-2.console.aws.amazon.com/sqlworkbench/home?region=ap-southeast-2#/account/configuration) for this lab.

**2.2 Create Tables**

Amazon Redshift is an ANSI SQL compliant data warehouse. You can create tables using familiar [CREATE TABLE](https://docs.aws.amazon.com/redshift/latest/dg/r_CREATE_TABLE_NEW.html) statements.

To create 8 tables from TPC Benchmark data model, copy the following create table statements and run them. Given below is the data model for the tables.

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

![image.png](/images/2/2-3.png)

**2.3 Data Loading**

In this lab, you will learn following approaches to load data into Redshift.

1. Load data from S3 to Redshift. You will use this process for loading 7 tables out of 8 tables created in above step.
2. Load data from your desktop file to Redshift. You will use this process for loading 1 table out of 8 created in above step i.e. nation table.
   
We split the load process to make you familiar with both the approaches.

- **Loading data from S3**
  
[COPY command](https://docs.aws.amazon.com/redshift/latest/dg/r_COPY.html) loads large amounts of data effectively from Amazon S3 into Amazon Redshift. A single COPY command can load from multiple files into one table. It automatically loads the data in parallel from all the files present in the S3 location you provide.

If the target table is empty, **COPY** Command can perform compression encoding for each column automatically based on the column datatype so that you need not worry about choosing the right compression algorithm for each column. To ensure that Redshift performs a compression analysis, we are going to set the **COMPUPDATE** parameter to **PRESET** in the **COPY** commands.

The sample data to load is made available in a public [Amazon S3 bucket](https://us-west-2.console.aws.amazon.com/s3/buckets/redshift-immersionday-labs?region=us-west-2&prefix=data%2F&showversions=false&bucketType=general).

Copy the following statements to load data into 7 tables.

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

The estimated time to load the data is as follows. While waiting for the copy process to finish, you can move to next section [Loading data from your local machine using QEV2](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab2#loading-data-from-your-local-machine-using-qev2).

- REGION (5 rows) - 2s
- CUSTOMER (15M rows) – 2m
- ORDERS - (76M rows) - 10s
- PART - (20M rows) - 2m
- SUPPLIER - (1M rows) - 10s
- LINEITEM - (303M rows) - 22s
- PARTSUPPLIER - (80M rows) - 15s
  
**Note:** A few key takeaways from the above COPY statements.

- **COMPUPDATE PRESET ON** will assign compression using the Amazon Redshift best practices related to the data type of the column but without analyzing the data in the table.

- **COPY** for the **REGION** table points to a specific file (region.tbl.lzo) while COPY for other tables point to a prefix to multiple files (lineitem.tbl.).

- **COPY** for the **SUPPLIER** table points a manifest file (supplier.json). A manifest file is a JSON file that lists the files to be loaded and their locations.

- **Loading data from your local machine using QEV2**
  
You will be loading the nation table through query editor v2 by importing the data from your local machine.

*+ One time setup*

As a one time setup, you need to create a S3 bucket in the same region as redshift instance. And add the S3 bucket path in QEV2 account setting page.

S3 bucket is already pre-created in this lab environment. Navigate to [AWS CloudFormation](https://ap-southeast-2.signin.aws.amazon.com/oauth?response_type=code&client_id=arn%3Aaws%3Asignin%3A%3A%3Aconsole%2Fcloudformation&redirect_uri=https%3A%2F%2Fconsole.aws.amazon.com%2Fcloudformation%2Fhome%3FhashArgs%3D%2523%26isauthcode%3Dtrue%26oauthStart%3D1723994246570%26state%3DhashArgsFromTB_ap-southeast-2_4edb45c164185361&forceMobileLayout=0&forceMobileApp=0&code_challenge=9xZ7-RKc3ZqaBRZs2hRfaMaNN99JIO-vOWYdGlBdM6M&code_challenge_method=SHA-256)  and click on 'cfn' stack. Under 'Outputs' tab note down the S3Bucket. You’ll need this value in next step.

![image.png](/images/2/2-5.png)

In QEV2 go to **Settings**(bottom left) --> **Account settings** --> **S3 bucket name**.

![image.png](/images/2/2-6.png)

![image.png](/images/2/2-7.png)

*+Load File*

You can download the sample [*Nation.csv*](https://redshift-immersionday-labs.s3.us-west-2.amazonaws.com/data/nation/Nation.csv) file onto your local machine using the below link.


Let us load the downloaded *Nation* file on your desktop into Redshift.

1. From **resources** section, select and connect to your serverless workgroup.

Note: The *Nation* table is already created in the [Create Tables](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab2#create-tables) step. However you can also create a table through user interface by following **Create**-->**Table** option in left navigation pane in *Query Editor v2*.

2. Click on **Load data** -- select **Load from local file** -- Browse and select the Nation.csv file downloaded earlier -- **Data conversion parameters** -- **Check Ignore header rows** -- **Next** -- **Table options** select your serverless work group -- **Databas**e = dev -- **Schema** = public -- **Table** = nation -- **Load data** -- Notification of**Loading data from a local file succeeded** would be shown on top.

![image.png](/images/2/2-8.png)

Note: **Machines with upload restrictions**

If your machine has restrictions that don't allow uploads to S3 you will not be able to load a local file. However, you can use the *Query Editor v* to load a file directly from S3. To do this, select the **Load from S3 bucket** option and input the following location:

`s3://redshift-immersionday-labs/data/nation/Nation.csv`

You will need to select the S3 location as `us-west-2`.

Follow the below instructions as-is. The only change is selecting the IAM role when choosing which table to load. There will be only one role available in **AWS Sponsered Events** (`RedshiftServerlessImmersionRole`).

![image.png](/images/2/2-9.png)

![image.png](/images/2/2-10.png)

**2.4 Load Validation**

Let us do a quick check of counts on few tables to ensure data is loaded as expected. You can open a new editor and run below queries if the copy process is still loading some tables.

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

**2.5 Troubleshooting Loads**

To troubleshoot any data load issues, you can query `SYS_LOAD_ERROR_DETAIL`.

In addition, you can validate your data without actually loading the table. You can use the `NOLOAD` option with the `COPY` command to make sure that your data will load without any errors before running the actual data load. Notice that running `COPY` with the `NOLOAD` option is much faster than loading the data since it only parses the files.

Let’s try to load the `CUSTOMER` table with a different data file with mismatched columns.

```jsx
COPY customer FROM 's3://redshift-immersionday-labs/data/nation/nation.tbl.'
iam_role default
region 'us-west-2' lzop delimiter '|' noload;
```

![image.png](/images/2/2-14.png)

You will get the following error.

```jsx
ERROR: Load into table 'customer' failed. Check 'sys_load_error_detail' system table for details.
```

Query the STL_LOAD_ERROR system table for details.

```jsx
select * from SYS_LOAD_ERROR_DETAIL;
```

![image.png](/images/2/2-15.png)

Notice that there is one row for the column c_nationkey with datatype int with an error message "Invalid digit, Value 'h', Pos 1" indicating that you are trying to load character into an integer column.

**2.6 Automatic Table Maintenance - ANALYZE and VACUUM**

- **Analyze:**

When loading into an empty table, the `COPY` command by default collects statistics (ANALYZE). If you are loading a non-empty table using `COPY` command, in most cases, you don't need to explicitly run the `ANALYZE` command. Amazon Redshift monitors changes to your workload and automatically updates statistics in the background. To minimize impact to your system performance, automatic analyze runs during periods when workloads are light.

If you need to analyze the table immediately after load, you can still manually run `ANALYZ`E` command.

To run `ANALYZE` on `orders` table, copy the following command and run it.

```jsx
ANALYZE orders;
```

![image.png](/images/2/2-16.png)

- **Vacuum:**

**+ Vacuum Delete:** When you perform delete on a table, the rows are marked for deletion(soft deletion), but are not removed. When you perform an update, the existing rows are marked for deletion(soft deletion) and updated rows are inserted as new rows. Amazon Redshift automatically runs a `VACUUM DELETE` operation in the background to reclaim disk space occupied by rows that were marked for deletion by `UPDATE` and `DELETE` operations, and compacts the table to free up the consumed space. To minimize impact to your system performance, automatic VACUUM `DELETE` runs during periods when workloads are light.

If you need to reclaim diskspace immediately after a large delete operation, for example after a large data load, then you can still manually run the `VACUUM DELETE` command. Lets see how `VACUUM DELETE` reclaims table space after delete operation.

First, capture `tbl_rows`(Total number of rows in the table. This value includes rows marked for deletion, but not yet vacuumed) and `estimated_visible_rows`(The estimated visible rows in the table. This value does not include rows marked for deletion) for the ORDERS table. Copy the following command and run it.

```jsx
select "table", size, tbl_rows, estimated_visible_rows
from SVV_TABLE_INFO
where "table" = 'orders';
```

![image.png](/images/2/2-17.png)

Next, delete rows from the ORDERS table. Copy the following command and run it.

```jsx
delete orders where o_orderdate between '1997-01-01' and '1998-01-01';
```

![image.png](/images/2/2-18.png)

Next, capture the tbl_rows and estimated_visible_rows for ORDERS table after the deletion.

Copy the following command and run it. Notice that the tbl_rows value hasn't changed, after deletion. This is because rows are marked for soft deletion, but VACUUM DELETE is not yet run to reclaim space.

```jsx
select "table", size, tbl_rows, estimated_visible_rows
from SVV_TABLE_INFO
where "table" = 'orders';
```

![image.png](/images/2/2-19.png)

Now, run the VACUUM DELETE command. Copy the following command and run it.

```jsx
vacuum delete only orders;
```

![image.png](/images/2/2-20.png)

Confirm that the VACUUM command reclaimed space by running the following query again and noting the tbl_rows value has changed.

```jsx
select "table", size, tbl_rows, estimated_visible_rows
from SVV_TABLE_INFO
where "table" = 'orders';
```

![image.png](/images/2/2-21.png)

**+ Vacuum Sort:**

Vacuum Sort: When you define **SORT KEYS**  on your tables, Amazon Redshift automatically sorts data in the background to maintain table data in the order of its sort key. Having sort keys on frequently filtered columns makes block level pruning, which is already efficient in Amazon Redshift, more efficient.

Amazon Redshift keeps track of your scan queries to determine which sections of the table will benefit from sorting and it automatically runs `VACUUM SORT` to maintain sort key order. To minimize impact to your system performance, automatic `VACUUM SORT` runs during periods when workloads are light.

`COPY` command automatically sorts and loads the data in sort key order. As a result, if you are loading an empty table using `COPY` command, the data is already in sort key order. If you are loading a non-empty table using `COPY` command, you can optimize the loads by loading data in incremental sort key order because `VACUUM SORT` will not be needed when your load is already in sort key order

For example, `orderdate` is the sort key on `orders` table. If you always load data into `orders` table where `orderdate` is the current date, since current date is forward incrementing, data will always be loaded in incremental sortkey(`orderdate`) order. Hence, in that case `VACUUM SORT` will not be needed for `orders` table.

If you need to run `VACUUM SORT`, you can still manually run it as shown below. Copy the following command and run it.

```jsx
vacuum sort only orders;
```

![image.png](/images/2/2-22.png)

**+ Vacuum Recluster:**

Use `VACUUM` recluster whenever possible for manual `VACUUM` operations. This is especially important for large objects with frequent ingestion and queries that access only the most recent data. `Vacuum recluster` only sorts the portions of the table that are unsorted and hence runs faster. Portions of the table that are already sorted are left intact. This command doesn't merge the newly sorted data with the sorted region. It also doesn't reclaim all space that is marked for deletion. 

In order to run vacuum recluster on orders, copy the following command and run it.

```jsx
vacuum recluster orders;
```

![image.png](/images/2/2-23.png)

**+ Vacuum Boost:**

`Boost` runs the `VACUUM` command with additional compute resources, as they're available. With the `BOOST` option, `VACUUM` operates in one window and blocks concurrent deletes and updates for the duration of the `VACUUM` operation. Note that running vacuum with the `BOOST` option contends for system resources, which might affect performance of other queries. As a result, it is recommended to run the `VACUUM BOOST` when the load on the system is light, such as during maintenance operations. 

In order to run vacuum recluster on orders table in boost mode, copy the following command and run it.

```jsx
vacuum recluster orders boost;
```

![image.png](/images/2/2-24.png)


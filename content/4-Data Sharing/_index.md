---
title : "Data Sharing"
date : "`r Sys.Date()`"
weight : 4
chapter : false
pre : " <b> 4. </b> "
---
**4. Data Sharing**

In this lab, you will walk through steps for data sharing between a serverless endpoint and a provisioned cluster.

**Contents**

- Before you begin
- Introduction
- Identify namespaces
- Create data share on producer
- Query data from consumer
- Create external schema in consumer
- Load local data and join to shared data
- Before you leave

**4.1 Before you begin**

This lab assumes that you have launched an Amazon Redshift Serverless endpoint and Provisioned cluster. If you have not already done so, please see [**Getting Started**](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab1) and follow the instructions there. We will use [Amazon Redshift QueryEditorV2](https://console.aws.amazon.com/sqlworkbench/home)  for this lab.

**4.2 Introduction**

**A datashare** *is the unit of sharing data in Amazon Redshift. Use datashares to share data in the same AWS account or different AWS accounts. Also, share data for read purposes across different Amazon Redshift clusters. Each datashare is associated with a specific database in your Amazon Redshift cluster.*

*Datashare objects are objects from specific databases on a cluster that producer cluster administrators can add to datashares to be shared with data consumers. Datashare objects are read-only for data consumers. Examples of datashare objects are tables, views, and user-defined functions. You can add datashare objects to datashares while creating datashares or editing a datashare at any time.*

**Reference:** [Amazon Redshift Data Sharing](https://aws.amazon.com/redshift/features/data-sharing/)

**4.3 Identify namespaces**

We will run simple SQL statements to find the namespace for both producer cluster and consumer cluster. These values will be used in subsequent instructions.

**NOTE:** namespace is globally unique identifier (GUID) automatically created during Amazon Redshift cluster creation and attached to the cluster. It is used to uniquely reference the Redshift cluster.

Run following command in query editor connecting to serverless endpoint (producer - Serverless: workgroup-xxxxxxx). Note down the namespace from output. Ensure that the editor is connected to the producer cluster.

```jsx
-- This should be run on the producer 
select current_namespace;
```
![image.png](/images/4/4-01.png)

Run following command in query editor connecting to the provisioned cluster (consumer - consumercluster-xxxxxxxxxx). Note down the namespace from output. Ensure that the editor is connected to the consumer cluster.

```jsx
-- This should be run on consumer 
select current_namespace;
```
![image.png](/images/4/4-012.png)

**4.4 Create data share on producer**

Run following commands connecting to the serverless endpoint (producer - Serverless: workgroup-xxxxxxx) to create a data share and add `customer` table to data share.

Replace `<<consumer namespace>>` with the consumer cluster namespace captured begining of this lab.

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

![image.png](/images/4/4-04.png)

**4.5 Query data from consumer**

Please run following commands in the provisioned cluster (consumer - consumercluster-xxxxxxxxxx) to create a local database on shared data.

**NOTE:** Replace `<<producer namespace>>` with the producer namespace captured begining of this lab.

```jsx
-- View shared objects
show datashares;
select * from SVV_DATASHARE_OBJECTS;

-- Create local database
CREATE DATABASE cust_db FROM DATASHARE cust_share OF NAMESPACE '<<producer namespace>';

-- Select query to check the count
select count(*) from cust_db.public.customer; -- count 15000000
```

![image.png](/images/4/4-05.png)

#### [OPTIONAL] 

**- Create external schema in consumer**

Running the following command in the provisioned consumer cluster (consumer - consumercluster-xxxxxxxxxx) will create an external schema called cust_db_public in your consumer's dev database. This is useful when you require all of your data to be available in a single database, simplifying SQL statements to use two-dot notation instead of three-dot (as shown in the next step). This step is also necessary when using some BI tools to allow metadata to be correctly refreshed in their UI. Creating an external schema using a datashare object can also simplify the process of splitting workloads to datasharing clusters. By creating external schemas in your consumer cluster with the same name as in the producer, workload queries can be migrated to the consumer cluster without needing any change.

```jsx
-- Create local schema
CREATE EXTERNAL SCHEMA cust_db_public
FROM REDSHIFT
DATABASE 'cust_db'
SCHEMA 'public';

-- Select query to check the count
select count(*) from cust_db_public.customer; -- count 15000000
```

![image.png](/images/4/4-6.png)

**Notice:** you get the same results as in the previous step, but the select statement is using the new external schema.

**- Load local data and join to shared data**

Run following commands in a query editor tab that is to the provisioned cluster (consumer - consumercluster-xxxxxxxxxx). The following commands are used to create and load orders table data that will be later joined with customer data shared from the producer endpoint.

```jsx
-- Create orders table in provisioned cluster (consumer).
DROP TABLE IF EXISTS orders;
create table orders
(  O_ORDERKEY bigint NOT NULL,  
O_CUSTKEY bigint,  
O_ORDERSTATUS varchar(1),  
O_TOTALPRICE decimal(18,4),  
O_ORDERDATE Date,  
O_ORDERPRIORITY varchar(15),  
O_CLERK varchar(15),  
O_SHIPPRIORITY Integer,  O_COMMENT varchar(79))
distkey (O_ORDERKEY)
sortkey (O_ORDERDATE);

-- Load orders table from public data set
copy orders from 's3://redshift-immersionday-labs/data/orders/orders.tbl.'
iam_role default region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

-- Select count to verify the data load
select count(*) from orders;  -- count 76000000
```

![image.png](/images/4/4-7.png)

Run the below BI query to find total sale price by customer segment and order priority joining locally loaded order data to shared customer data.

```jsx
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM cust_db.public.customer c
JOIN orders o on c_custkey = o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

![image.png](/images/4/4-8.png)

If you created the external schema in the previous step, run the below query using the local schema instead of the datashare database. This will provide the same result as the query above.

```jsx
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM cust_db_public.customer c
JOIN orders o on c_custkey = o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

![image.png](/images/4/4-9.png)
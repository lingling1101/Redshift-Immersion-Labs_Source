---
title : "Operations"
date :  "`r Sys.Date()`" 
weight : 7
chapter : false
pre : " <b> 7. </b> "
---

**VII. Operations**

In this lab, you will go through the monitoring (from both console and system views),audit logging features, changing base RPU setting and limits.

**Contents**

- Before you begin
- Monitoring
- System views
- Audit logging
- Changing RPUs and setting limits

**7.1 Before you begin**

This lab assumes that you have launched an **Amazon Redshift Serverless** endpoint. If you have not already done so, please see [Getting Started](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab1) and follow the instructions there. We will use **Amazon Redshift [QueryEditorV2](https://console.aws.amazon.com/sqlworkbench/home)** and the [**Amazon Redshift Serverless**](https://console.aws.amazon.com/redshiftv2/home?#serverless-dashboard)  for this lab.

**7.2 Monitoring**

In this section, you will go through various monitoring options and set filters to experience the functionality. You will use the console to see query history, database performance and resource usage.

Once you’re in the **Redshift Serverless** dashboard, navigate to [**Query and database monitoring**](https://console.aws.amazon.com/redshiftv2/home?#serverless-query-and-database-monitoring) on the left-hand side menu.

- Select the **workgroup-xxxxxxx** workgroup
- Expand **Additional filtering** options. You can filter these metrics on several categories. Feel free to modify these values based on workload patterns that you are most interested in deciphering. For this lab, set the values as show in the screenshot - these are defaults to start with.

![image.png](/images/7/7-1.png)

You will not be able to see queries yet. You need to grant monitoring access to your console role. This is a pre-requisite step.

Login as superuser (awsuser) in Query Editor and run below SQL.

```jsx

-- Run below command if you are running this event from workshop
grant role sys:monitor to "IAMR:WSParticipantRole";

-- Run below command if you are running this event from Event Engine
grant role sys:monitor to "IAMR:TeamRole" 
```

- Now navigate back to the [**Query and database monitoring**](https://console.aws.amazon.com/redshiftv2/home?#serverless-query-and-database-monitoring)  section on the left-hand side menu and change few filters to start seeing the queries.

- Scroll below, you can see your query run-times for selected time interval. Use this graph to look into query concurrency as well as to research more into certain queries that took longer to execute than expected.

![image.png](/images/7/7-2.png)

- Scroll again below to view the Queries and loads chart. Here you can see all the completed, running, and aborted queries.

![image.png](/images/7/7-3.png)

Navigate to the **Database Performance** tab to view:

- **Queries completed per second**: The average number of queries completed per second

![image.png](/images/7/7-4.png)

- **Queries duration**: The average amount of time to complete a query

![image.png](/images/7/7-5.png)

- **Database connections**: The average number of active database connections

![image.png](/images/7/7-6.png)

- **Running and Queued queries**

![image.png](/images/7/7-7.png)

Navigate to the **Resource Monitoring** section on the left navigation pane.

- Select the **default workgroup**
- Expand **Additional filtering** options. Choose **1 minute** time interval and review results. You can also try different ranges to see the results.

![image.png](/images/7/7-8.png)

- **RPU Capacity Used** - No.of RPUs consumed

![image.png](/images/7/7-9.png)

- **Compute usage** - RPU seconds

![image.png](/images/7/7-10.png)

**7.3 System Views**

Below system views in **Amazon Redshift Serverless** are used to monitor query and workload usage. These views are located in the **pg_catalog** schema. These system views have been designed to give you the information needed to monitor Amazon Redshift Serverless, which are much simpler than various system views available for provisioned clusters.

- SYS_QUERY_HISTORY
- SYS_QUERY_DETAIL
- SYS_EXTERNAL_QUERY_DETAIL
- SYS_LOAD_HISTORY
- SYS_LOAD_ERROR_DETAIL
- SYS_UNLOAD_HISTORY
- SYS_SERVERLESS_USAGE

Please refer to any new additions at [https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-monitoring.html](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-monitoring.html)

We will go through some of the frequently used system monitoring queries.

- Run below query to find the recent 10 completed, running and queued queries

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

- Run below query to show serverless usage summary including how much compute capacity is used to process queries and the amount of **Amazon Redshift** managed storage used.

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

In the result, you can see the periods of idle time where the cluster is auto paused and auto resumed when the queries starts coming in. You will not be charged when the cluster is paused.

- Query to show the loaded rows, bytes, tables and data source of copy commands

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

**7.4 Audit logging**

You can configure Amazon Redshift Serverless to export connection, user, and user-activity log data to a log group in Amazon CloudWatch Logs when creating workgroups

The first step in the process is to make sure audit logging has been enabled for the workgroup.

- Navigate to **Namespace**→**security and encryption**. Verify audit logging is on.

![image.png](/images/7/7-14.png)

![image.png](/images/7/7-15.png)

- Select the option **Edit** if Audit logging is disabled and check mark all the 3 options as shown below.

![image.png](/images/7/7-16.png)

- Now run following commands in **query editor** connecting to serverless default workgroup

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

You can monitor events in the Amazon CloudWatch Logs.

- Navigate to **Amazon CloudWatch** service and select **Metrics**→**All metrics** on the left navigation menu.

- Select **AWS/Redshift-Serverless** to get details on Serverless usage. Below is a example screen.

![image.png](/images/7/7-20.png)

Amazon Redshift Serverless metrics are divided into groups of compute metrics, data and storage metrics. Select **Workgroup*** to get compute seconds details.

![image.png](/images/7/7-21.png)

Select your workgroup to get details on compute resources for the specific workgroup. see below example screen.

![image.png](/images/7/7-22.png)

Select **DatabaseName**, **Namespace** to get details on storage resources for a specific namespace as shown below.

![image.png](/images/7/7-23.png)

Select **Dev** database for your namespace to find out total number of tables as shown below. Please ensure you de-select all others to see a graph similar to below.

![image.png](/images/7/7-24.png)

You can also export the CloudWatch log groups events to S3. Please see below example screen.

**NOTE**: You may not be able to access this in lab environment. But you can try in your own environment.

![image.png](/images/7/7-25.png)

**7.5 Changing RPUs and Setting limits**

- Select your default workgroup, click on the **Limits** tab.
- Click on **Edit** in **Base capacity in Redshift processing units (RPUs)** section.

![image.png](/images/7/7-26.png)

By default, when you launch your serverless endpoint, you’re provisioned with 128 RPUs. Here you can adjust the **Base capacity** setting from 32 RPUs to 512 RPUs in the incremental units of 8 RPUs. Each RPU, which is a unit of compute resource with 16 GB memory.

For this lab, you can just leave it at 128 RPUs. Select **Cancel**.

- Scroll down to **Usage limits** and click on **Manage usage limits**.

![image.png](/images/7/7-27.png)

- Click on **Add limit**.

![image.png](/images/7/7-28.png)

In this tab, you can configure usage capacity limits to curb your overall Redshift Serverless bill. To control usage, set the maximum RPU-hours by frequency.

- Set the **Frequency** to **Daily**, **Usage limit** (hours) to **1**, and **Action** to **Turn off user queries**.
  
(optional) You can set email alerts by selecting **Create SNS topic** to be notified once usage limit as been met. You can park this for home assignment.

- Once finished, click **Save changes**. After 1 RPU hour of daily usage, your users will no longer be able to send queries to Redshift Serverless and stops any RPU billing.

![image.png](/images/7/7-29.png)

- Run the below sample query few times. After few times, it should get usage limit reached error message.

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

- You should get an error: `ERROR: Query reached usage limit*` once you reached the limit and an email notification will be sent if you subscribed through SNS topic.

![image.png](/images/7/7-31.png)

Go back to **Manage usage limits** and delete the limit before proceeding to the next lab. If you miss this step, you will not able to run queries further.



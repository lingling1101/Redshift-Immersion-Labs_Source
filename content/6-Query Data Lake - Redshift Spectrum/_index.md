---
title : "Query Data Lake - Redshift Spectrum"
date :  "`r Sys.Date()`" 
weight : 6 
chapter : false
pre : " <b> 6. </b> "
---
### **6. Query Data Lake - Redshift Spectrum**

In this lab, we show you how to query data in your Amazon S3 data lake with Amazon Redshift without loading or moving data. We will also demonstrate how you can leverage views which union data in Redshift Managed storage with data in S3. You can query structured and semi-structured data from files in Amazon S3 without having to copy or move data into Amazon Redshift tables. For latest guide on the file types that can be queried with Redshift Spectrum, please refer to supported data formats.

![image.png](/images/6/6-001.png)

### **Contents**

- Before you begin
- Use-case description
- Instructions
- Before you leave

### **6.1 Before you begin**

This lab assumes you have launched a Redshift Serverless Warehouse. If you have not created Redshift Serverless warehouse see [Getting Started](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab1). We will use the Redshift Query Editor V2 in the Redshift console for this lab.

Please find your region by following the image below and select s3 data sets as per the instructions for your region.

> This lab requires a Redshift Serverless namespace in us-east-1(N. Virginia) or us-west-2(Oregon) or eu-west-1(Ireland) or ap-northeast-1(Tokyo) regions as the data in s3 is located in these four regions.

### **6.2 Use-Case description**

**Objective**: Derive data insights to showcase the effect of blizzard on number of taxi rides in January 2016.

**Data set description**: NY city taxi trip data including number of taxi rides by year and month for 3 different taxi companies - fhv, green, and yellow.

**Data set S3 location**:

+ **us-east-1** region - [https://s3.console.aws.amazon.com/s3/buckets/redshift-demos?region=us-east-1&prefix=data/NY-Pub/](https://s3.console.aws.amazon.com/s3/buckets/redshift-demos?region=us-east-1&prefix=data/NY-Pub/) 

+ **us-west-2** region - [https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics?prefix=canonical/NY-Pub/](https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics?prefix=canonical/NY-Pub/) 

+ **eu-west-1** region - [https://s3.console.aws.amazon.com/s3/buckets/redshift-demos-dub?region=eu-west-1&prefix=NY-Pub/](https://s3.console.aws.amazon.com/s3/buckets/redshift-demos-dub?region=eu-west-1&prefix=NY-Pub/) 

+ **ap-northeast-1** region - [https://s3.console.aws.amazon.com/s3/buckets/redshift-demos-nrt?region=ap-northeast-1&prefix=NY-Pub/](https://s3.console.aws.amazon.com/s3/buckets/redshift-demos-nrt?region=ap-northeast-1&prefix=NY-Pub/) 

Below is an overview of the use case steps involved in this lab.
![image.png](/images/6/6-2.png)

### **6.3 Instructions**

### **1. Create and run Glue crawler to populate Glue data catalog**

In this part of the lab, we will perform following activities:

- Query historical data residing on S3 by creating an external DB for Redshift Spectrum.
- Introspect the historical data, perhaps rolling-up the data in novel ways to see trends over time, or other dimensions.
- Note the partitioning scheme is Year, Month, Type (where Type is a taxi company). Here's a Screenshot: [https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics/canonical/NY-Pub/](https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics/canonical/NY-Pub/)

> Region
> 
> Clicking the above link may change your default region. When continuing to the next steps make sure that your region is correct before creating the Glue Crawler.

![image.png](/images/6/6-03.png)

**Create external schema (and DB) for Redshift Spectrum**

You can create an external table in Amazon Redshift, AWS Glue, Amazon Athena, or an Apache Hive metastore. If your external table is defined in AWS Glue, Athena, or a Hive metastore, you first create an external schema that references the external database. Then, you can reference the external table in your SELECT statement by prefixing the table name with the schema name, without needing to create the table in Amazon Redshift.

In this lab, you will use AWS Glue Crawler to create external table **adb305.ny_pub** stored in **parquet** format in s3 location for that region.

**+ Navigate to the Glue Crawler Page:**

- **us-east-1** region - [https://us-east-1.console.aws.amazon.com/glue/home](https://us-east-1.console.aws.amazon.com/glue/home) 

- **us-west-2** region - [https://us-west-2.console.aws.amazon.com/glue/home](https://us-west-2.console.aws.amazon.com/glue/home) 

- **eu-west-1** region - [https://eu-west-1.console.aws.amazon.com/glue/home](https://eu-west-1.console.aws.amazon.com/glue/home) 

- **ap-northeast-1** region - [https://ap-northeast-1.console.aws.amazon.com/glue/home](https://ap-northeast-1.console.aws.amazon.com/glue/home)

![image.png](/images/6/6-4.png)

**+** Click on **Create Crawler**, and enter the crawler name **NYTaxiCrawler** and click **Next**.

![image.png](/images/6/6-5.png)

**+** Click on **Add a data source**.

![image.png](/images/6/6-6.png)

**+** Choose **S3** as the data store.

**+** Select **In a different account**

**+** Enter S3 file path -

- For **us-east-1** region - `s3://redshift-demos/data/NY-Pub/`

- For **us-west-2** region - `s3://us-west-2.serverless-analytics/canonical/NY-Pub/`

- For **eu-west-1** region - `s3://redshift-demos-dub/NY-Pub/`

- For **ap-northeast-1** region - `s3://redshift-demos-nrt/NY-Pub/`

**+** Click on **Add an S3 data source**.

![image.png](/images/6/6-7.png)

**+** Click **Next**

![image.png](/images/6/6-8.png)

**+** Click **Create new IAM role** and click **Next**

![image.png](/images/6/6-9.png)

**+** Enter **AWSGlueServiceRole-RedshiftImmersion** and click **Create**

![image.png](/images/6/6-10.png)

**+** Click on **Add database** and enter Name **spectrumdb**

![image.png](/images/6/6-11.png)

![image.png](/images/6/6-012.png)

**+** Go back to **Glue Console**, refresh the target database and select **spectrumdb**

![image.png](/images/6/6-13.png)

**+** Select all remaining defaults and click **Create crawler**. Select the crawler - **NYTaxiCrawler** and click **Run**.

![image.png](/images/6/6-14.png)

**+** After Crawler run completes, you can see a new table **ny_pub** in **Glue Catalog**

![image.png](/images/6/6-015.png)

### **2. Create external schema adb305 in Redshift and select from Glue catalog table - ny_pub**

**+** Go to **Redshift console**.

- **us-east-1** region - [https://us-east-1.console.aws.amazon.com/redshiftv2/home?region=us-east-1#/serverless-setup](https://us-east-1.console.aws.amazon.com/redshiftv2/home?region=us-east-1#/serverless-setup)

- **-west-2** region - [https://us-west-2.console.aws.amazon.com/redshiftv2/home?region=us-west-2#/serverless-setup](https://us-west-2.console.aws.amazon.com/redshiftv2/home?region=us-west-2#/serverless-setup)

- **eu-west-1** region - [https://eu-west-1.console.aws.amazon.com/redshiftv2/home?region=eu-west-1#/serverless-setup](https://eu-west-1.console.aws.amazon.com/redshiftv2/home?region=eu-west-1#/serverless-setup)

- **ap-northeast-** region - [https://ap-northeast-1.console.aws.amazon.com/redshiftv2/home?region=ap-northeast-1#/serverless-setup](https://ap-northeast-1.console.aws.amazon.com/redshiftv2/home?region=ap-northeast-1#/serverless-setup)

Click on **Serverless dashboard** menu item to the left side of the console. Click on the name space provisioned earlier. Click **Query data**.

![image.png](/images/6/6-016.png)

- Create an external schema **adb305** pointing to your **Glue Catalog Database spectrumdb**.

```jsx
CREATE external SCHEMA adb305
FROM data catalog DATABASE 'spectrumdb'
IAM_ROLE default
CREATE external DATABASE if not exists;
```

![image.png](/images/6/6-017.png)

**Pin-point the Blizzard**

You can query the table **ny_pub**, defined in **Glue Catalog** from **Redshift** external schema. In January 2016, there is a date which had the lowest number of taxi rides due to a blizzard. Can you find that date?

```jsx
SELECT TO_CHAR(pickup_datetime, 'YYYY-MM-DD'),COUNT(*)
FROM adb305.ny_pub
WHERE YEAR = 2016 and Month = 01
GROUP BY 1
ORDER BY 2;
```

![image.png](/images/6/6-18.png)

### **3. Create internal schema workshop_das**

Create a schema **workshop_das** for tables that will reside on the **Redshift Managed Storage**.

```jsx
CREATE SCHEMA workshop_das;
```

![image.png](/images/6/6-19.png)

### **4. Run CTAS to create and load Redshift table workshop_das.taxi_201601 by selecting from external table**

Create table **workshop_das.taxi_201601** to load data for **green** taxi company for January 2016

```jsx
CREATE TABLE workshop_das.taxi_201601 AS
SELECT *
FROM adb305.ny_pub
WHERE year = 2016 AND month = 1 AND type = 'green';
```

![image.png](/images/6/6-20.png)

> **Note**: What about column compression/encoding? Remember that on a CTAS, Amazon Redshift automatically assigns compression encoding as follows:

- Columns that are defined as sort keys are assigned RAW compression.
- Columns that are defined as BOOLEAN, REAL, or DOUBLE PRECISION, or GEOMETRY data types are assigned RAW compression.
- Columns that are defined as SMALLINT, INTEGER, BIGINT, DECIMAL, DATE, TIMESTAMP, or TIMESTAMPTZ are assigned AZ64 compression.
- Columns that are defined as CHAR or VARCHAR are assigned LZO compression.

[https://docs.aws.amazon.com/redshift/latest/dg/r_CTAS_usage_notes.html](https://docs.aws.amazon.com/redshift/latest/dg/r_CTAS_usage_notes.html)

```jsx
ANALYZE COMPRESSION workshop_das.taxi_201601;
```

![image.png](/images/6/6-021.png)

- Add to the **taxi_201601** table with an **INSERT/SELECT** statement for other taxi companies.

```jsx
INSERT INTO workshop_das.taxi_201601 (
SELECT *
FROM adb305.ny_pub
WHERE year = 2016 AND month = 1 AND type != 'green');
```

![image.png](/images/6/6-22.png)

### **5. Drop 201601 partitions from external table**

Now that we've loaded all January, 2016 data, we can remove the partitions from the Spectrum table so there is no overlap between the **Redshift Managed Storage (RMS)** table and the **Spectrum** table.

```jsx
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='yellow');
```

![image.png](/images/6/6-23.png)

### **6. Create combined view public.adb305_view_NY_TaxiRides**

```jsx
CREATE VIEW adb305_view_NYTaxiRides AS
  SELECT * FROM workshop_das.taxi_201601
  UNION ALL
  SELECT * FROM adb305.ny_pub
WITH NO SCHEMA BINDING;
```

![image.png](/images/6/6-024.png)

**Explain displays the execution plan for a query statement without running the query**

- Note the use of the partition columns in the SELECT and WHERE clauses. Where were those columns in your Spectrum table definition?
- Note the filters being applied either at the partition or file levels in the Spectrum dataset of the query (versus the Redshift Managed Storage dataset).
- If you actually run the query (and not just generate the explain plan), does the runtime surprise you? Why or why not?

```jsx
EXPLAIN
SELECT year, month, type, COUNT(*)
FROM adb305_view_NYTaxiRides
WHERE year = 2016 AND month IN (1,2) AND passenger_count = 4
GROUP BY 1,2,3 ORDER BY 1,2,3;
```

![image.png](/images/6/6-25.png)

![image.png](/images/6/6-026.png)

Note the **S3 Seq Scan** was run against the data on Amazon S3. The **S3 Seq Scan** node shows the Filter: (passenger_count = 4) was processed in the **Redshift Spectrum** layer.

For ways to improve Redshift Spectrum performance, please refer to [https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-performance.html](https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-performance.html)
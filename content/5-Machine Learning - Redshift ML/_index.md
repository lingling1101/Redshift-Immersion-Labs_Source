---
title : "Machine Learning - Redshift ML"
date :  "`r Sys.Date()`" 
weight : 5 
chapter : false
pre : " <b> 5. </b> "
---
**V. Machine Learning - Redshift ML**

In this lab you will create a model using Redshift ML Auto.

**Contents**

- Before you begin
- Data preparation
- Create model
- Check accuracy and run inference query
- Explainability
- Before you leave
  
**5.1 Before you begin**

This lab assumes that you have launched an Amazon Redshift Serverless endpoint. If you have not already done so, please see [**Getting Started**](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab1) and follow the instructions there. We will use [Amazon Redshift QueryEditorV2](https://console.aws.amazon.com/sqlworkbench/home)  for this lab.

**5.2 Data preparation**

The Bank Marketing data set contains direct marketing campaigns of a Portuguese banking institution. The marketing campaigns were based on phone calls. Often, more than one contact to the same client was required, in order to assess (evaluate) if the product (bank term deposit) would be ('yes') or not ('no') subscribed.

The data set consists of the following attributes. The classification goal is to predict if the client will subscribe to a term deposit.

**Attributes for bank client data:**

- age (numeric)
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

**Reference:** [Amazon Redshift Data Sharing](https://archive.ics.uci.edu/ml/datasets/bank+marketing)

Execute the following statements to create and load the training table in Redshift. Note the additional field y which will contain yes or no indicating the result of the term deposit subscription. The training data will be loaded with historical data and is used to create the model.

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

![image.png](/images/5/5-01.png)

Execute the following statements to create and load the inference table in Redshift. This data will be used to simulate new data which we can test against the model.

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

![image.png](/images/5/5-02.png)

**Create S3 bucket**

Before you create a model, you need to create a S3 bucket for storing intermediate results. Navigate to S3 and create a testanalytics bucket suffixed with your name, a random number, or you account number to give a unique name. You can find your account number on top right hand corner of management console. The intention is to create a S3 bucket with unique name.

![image.png](/images/5/5-3.png)

![image.png](/images/5/5-04.png)

**5.3 Create model**

Complete Autopilot generated with minimal user inputs. This will be a binary classification problem but auto pilot will choose the relevant algorithm based on the data and inputs

Execute the following statement to create the model. Replace `<<S3 bucket>>` with the bucket name you created above.

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

Run following command to check status of the model. It will be in TRAINING state. The create model will take ~ 60 minutes to run.

```jsx
show model model_bank_marketing;
```

![image.png](/images/5/5-06.png)

> As the training takes ~60minutes, you can move to next lab or a presentation. Please come back after an hour and execute rest of the steps.

**5.4 Check accuracy and run inference query**

Hope you gave enough time (~60min) for model to complete training. Run the same SQL Query as above to check status of the model. The model state should be in 'Ready' for you to proceed. Pay attention to the validation:f1 score - it will be between 0 and 1, the closer to 1, the better the model.

```jsx
show model model_bank_marketing;
```

![image.png](/images/5/5-07.png)

Check Inference/Accuracy of the model. Run the following queries - the first checks the accuracy of the model and the second will use the function created by the pre built model for the inference and against the data set in inference table bank_details_inference.

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

**5.5 Explainability**

You can identify which attributes are positively contributing to the prediction and how much is the contribution score by generating model explainability report. Please run below command to see model explainability.

```jsx
SELECT explain_model('model_bank_marketing');
```

![image.png](/images/5/5-010.png)

You will get a message stating that model did not train long enough to generate explainability report. That is expected as the model trained with MAX_RUNTIME of 3600 seconds. Usually you should increase the `MAX_RUNTIME` to 9600 or above for Redshift to generate explainability report. That gives enough to complete the explainability report steps model training.



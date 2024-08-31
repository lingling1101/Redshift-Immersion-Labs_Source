---
title : "Getting Started"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 1. </b> "
---
**I. Getting started**

Welcome to Redshift Immersion Day!!

You will be getting hands on experience on key Redshift features as part of this immersion day. It covers the foundational features such as data loading, transformation, data sharing, redshift spectrum, machine learning and operations. For advanced labs, you should use Redshift Deep Dive  workshop.

All the labs are designed to run on Redshift new serverless feature. Amazon Redshift Serverless  makes it simple to run and scale analytics without having to manage the instance type, instance size, lifecycle management, pausing, resuming, and so on. It automatically provisions and intelligently scales data warehouse compute capacity to deliver fast performance for even the most demanding and unpredictable workloads, and you pay only for what you use. Just load your data and start querying right away in the Amazon Redshift Query Editor or in your favorite business intelligence (BI) tool and continue to enjoy the best price performance and familiar SQL features in an easy-to-use, zero administration environment.

Below diagram shows the key features you will be working with and how they are used in typical data warehouse environment.

![image.png](/images/1/1-1.png)

**The AWS resources needed to run these labs include:**

- An Amazon Redshift Serverless endpoint in a new networking environment that includes a VPC, 3 Subnets, an Internet Gateway, and a Security Group to enable local access to your cluster.
- An Amazon Redshift Provisioned cluster in the same networking environment for use in the data sharing lab.
- A default role attached to your environments. To learn more about the authorizing Amazon Redshift to access other AWS Services, see the [Creating an IAM role as default for Amazon Redshift](https://docs.aws.amazon.com/redshift/latest/mgmt/default-iam-role.html).

**1.1 AWS-Sponsored Immersion Day**

If you are conducting these labs as part of an AWS-Sponsored Immersion Day, a lab account has been created with the above resources. For instructions on how to connect to your environment, see [Join Workshop Event](https://catalog.us-east-1.prod.workshops.aws/workshops/9f29cdba-66c0-445e-8cbb-28a092cb5ba7/en-US/lab0).

**1.2 Self-paced Immersion Day**

If you are conducting these labs on your own, use the following CFN template to launch the resources needed in your AWS account.

[Launch Stack](https://console.aws.amazon.com/cloudformation/home?#/stacks/new?stackName=RedshiftImmersionLab&templateURL=https://s3-us-west-2.amazonaws.com/redshift-immersionday-labs/immersionserverless.yaml)


![image.png](/images/1/1-2.png)

![image.png](/images/1/1-3.png)

**1.3 Configure client tool - Query editor V2**

Once you have access to an account with the required resources, let's validate you can connect to your Redshift environments. These labs utilize the Redshift web-based Query Editor v2. Navigate to the [Query editor v2](https://ap-southeast-1.console.aws.amazon.com/sqlworkbench/home?region=ap-southeast-1#/client).

If prompted, you may need to configure the Query Editor.
![image.png](/images/1/1-4.png)

On the left-hand side, click on the Redshift environment you want to connect to.
![image.png](/images/1/1-5.png)

A pop-up window should have opened.

Enter the Database name and user name. Click connect. These credentials should be used for both the Serverless endpoint (workgroup-xxxxxxx) as well as the provisioned cluster (consumercluster-xxxxxxxxxx).


```jsx
User Name: awsuser  
Password: Awsuser123
```
![image.png](/images/1/1-6.png)

**1.4 Run sample query**

Run the following query to list the users within the redshift cluster.

```jsx
select * from pg_user;
```

If you receive the following results, you have established connectivity and this lab is complete.
![image.png](/images/1/1-7.png)
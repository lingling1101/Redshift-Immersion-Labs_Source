---
title : "Clean up resources"
date :  "`r Sys.Date()`" 
weight : 8
chapter : false
pre : " <b> 8. </b> "
---

### **8. Clean up resources**

### **8.1 Delete Stacks**

- Go to CloudFormation
- Select the 'RedshiftImmersionLab' Stack
- Delete

![image.png](/images/8/8-1.png)

### **8.2 Delete Crawlers**
 
- Go to AWS Glue
- Go to the Crawlers section
- Select 'NYTaxiCrawlers'
- Choose Delete crawler
- Click Delete
  
![image.png](/images/8/8-2.png)

### **8.3 Delete Tables**

- Go to AWS Glue
- Go to the Tables section
- Select 'ny-pub'
- Choose Delete

![image.png](/images/8/8-3.png)

### **8.4 Delete S3 buckets**

- Click on the 'tesanalytics-11012003' bucket

![image.png](/images/8/8-4.png)

- Select Empty
  
![image.png](/images/8/8-5.png)

- Click Exit
- Select 'tesanalytics-11012003' again, then click Delete

![image.png](/images/8/8-6.png)





# building-your-first-big-data-application-on-aws

![dmslogo](http://d0.awsstatic.com/Solutions/Big%20Data/Big%20Data%20Redesign/BigData_Get-Started2.png)Lab Guide : Building your first Big Data application on AWS
==================================================================================================================================================================

*   The Velocity , Volume and Variety of data we are processing on a day to basis has drastically increased over time.
*   In this lab, we will create an simple application big data application on AWS which will be able to scale when needed.

### So what are we doing do ?

Collect

1.  Ingest logs into Amazon Kinesis Firehose Delivery Stream
2.  Store these log file in S3
3.  Launch an EMR cluster > Use spark to transform the unstructured logs files to desired format
4.  Use Hive with Spark on Amazon EMR to move this data to Amazon S3
5.  Launch a Amazon Redshift cluster > Load data in parallel into Amazon Redshift from Amazon S3
6.  Visualize using tool of your choice

![enter image description here](http://d0.awsstatic.com/Solutions/Big%20Data/Big%20Data%20Redesign/Big-Data-Redesign_Diagram_Enterprise-Data-Warehouse.png)

* * *

### Real-time Big Data Analytics - Amazon Kinesis Firehose

*   Easily load massive volumes of streaming data into AWS. Enable near real-time big data analytics with existing BI tools and dashboards you’re using today. [Learn more](http://aws.amazon.com/kinesis/firehose/)

### ![enter image description here](http://d0.awsstatic.com/Solutions/Big%20Data/Big%20Data%20Redesign/BigData_Object-Storage.png)Big Data Storage - Object Storage Amazon S3

*   Amazon S3 provides developers and IT teams with a highly reliable, secure, and scalable object storage for all your data, big or small. [Learn more](http://aws.amazon.com/s3/)

### ![enter image description here](http://d0.awsstatic.com/Solutions/Big%20Data/Big%20Data%20Redesign/BigData_Hadoop.png)Big Data Analytic Framework - Hadoop & Spark on Amazon EMR

*   Easily provision a fully managed Hadoop framework in minutes. Scale your Hadoop cluster dynamically and pay only for what you use. Run popular frameworks such as Apache Spark, Apache Tez, and Presto. [Learn more](http://aws.amazon.com/elasticmapreduce/)

### ![enter image description here](http://d0.awsstatic.com/Solutions/Big%20Data/Big%20Data%20Redesign/BigData_Redshift.png)Data Warehousing

*   Fully managed, petabyte-scale, data warehouse
*   Easily provision, configure and deploy a data warehouse within minutes. Amazon Redshift handles all the work needed to manage, monitor and scale it. Query & analyze big data for less than $1,000 per TB per year. [Learn more](http://aws.amazon.com/redshift/)

### ![enter image description here](http://d0.awsstatic.com/Solutions/Big%20Data/Big%20Data%20Redesign/BigData_QuickSight.png)Business Intelligence - Amazon QuickSight

*   Fast, cloud-powered BI for 1/10th the cost of traditional solutions
*   Deliver rich BI functionality to everyone in your organization. Enable employees to easily build visualizations, perform ad-hoc analysis, and quickly get business insights from big data. Perform advanced calculations and render visualizations rapidly. [Learn more](http://aws.amazon.com/quicksight/)  
    ![enter image description here](http://d0.awsstatic.com/Solutions/Big%20Data/Big%20Data%20Redesign/QS_SPICEngine.png)

* * *

Prerequisites
-------------

*   Create a IAM user with administrator privileges & setup your AWS CLI  
    using this user  
    *   [Installing the AWS Command Line Interface](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)
    *   [Configuring the AWS Command Line Interface](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html)
*   [Install latest version of boto3](https://boto3.readthedocs.io/en/latest/guide/quickstart.html) (AWS Python SDK)
*   Install [SQLWorkbenchJ](http://www.sql-workbench.net/downloads.html)
*   Understanding of security groups and ability to troubleshoot connectivity to servers / databases

* * *

### create a new s3 bucket for this lab

    aws --profile aws s3 mb s3://YOUR-S3-BUCKET
    

\*

### create a file : firehose-policy.json

    {
      "Version": "2012-10-17",
      "Statement": {
        "Effect": "Allow",
        "Principal": {"Service": "firehose.amazonaws.com"},
        "Action": "sts:AssumeRole"
      }
    }
    

\*

### create a file : s3-rw-policy.json

    { "Version": "2012-10-17", 
      "Statement": {
          "Effect": "Allow", 
          "Action": "s3:*", 
          "Resource": [
            "arn:aws:s3:::YOUR-S3-BUCKET-NAME", 
            "arn:aws:s3:::YOUR-S3-BUCKET-NAME/*"
          ]
      }
    }
    

### create a new role for kinesis firehose

    aws --profile aws iam create-role --role-name firehose-demo \
    --assume-role-policy-document file://firehose-policy.json
    
    aws --profile aws iam put-role-policy --role-name firehose-demo \
    --policy-name firehose-s3-rw \
    --policy-document file://s3-rw-policy.json
    

### Create a Firehose stream for incoming log data

Replace YOUR-FIREHOSEROLE-ARN with ARN from previous command

    aws --profile aws firehose create-delivery-stream \
      --delivery-stream-name demo-firehose-stream \
      --s3-destination-configuration \
    RoleARN=YOUR-FIREHOSEROLE-ARN,\
    BucketARN="arn:aws:s3:::YOUR-S3-BUCKET",\
    Prefix=firehose\/,\
    BufferingHints={IntervalInSeconds=60},\
    CompressionFormat=GZIP
    

### Launch an Amazon EMR cluster with Spark and Hive

    aws --profile aws emr create-cluster \
      --name "demo" \
      --release-label emr-4.5.0 \
      --instance-type m3.xlarge \
      --instance-count 2 \
      --ec2-attributes KeyName=YOUR-SSH-KEY \
      --use-default-roles \
      --applications Name=Hive Name=Spark Name=Zeppelin-Sandbox
    

### Create a single-node Amazon Redshift data warehouse

    aws --profile aws redshift create-cluster \
      --cluster-identifier demo \
      --db-name demo \
      --node-type dc1.large \
      --cluster-type single-node \
      --master-username master \
      --master-user-password Redshift123 \
      --publicly-accessible \
      --port 8192
    

### Execute this python code locally

    import boto3
    iam = boto3.client('iam')
    firehose = boto3.client('firehose')
    with open('weblog', 'r') as f:
        for line in f:
            firehose.put_record(
                    DeliveryStreamName='demo-firehose-stream',
                    Record={'Data': line}
                    )
            print 'Record added'
    

### View the Output Files in Amazon S3

    aws --profile aws s3 ls s3://YOUR-S3-BUCKET/firehose/ --recursive
    

### Connect to Your EMR Cluster and Zeppelin

*   Here you are tunneling the local port 8890 to remote port  
    ssh -i ~/.ssh/YOUR-SSH-KEY.pem -L 18890:localhost:8890 \\  
    hadoop@YOUR-EMR-CLUSTER-ADDRESS

### Open Zeppelin with your local web browser and create a new “Note”:

*   [http://localhost:18890](http://localhost:18890)
*   Import the Zeppelin notebook
*   Execute each paragraph in the notebook individually

### View the newly transformed files in S3

    aws --profile aws s3 ls s3://YOUR-S3-BUCKET/access-log-processed/ --recursive
    

### Connect to your newly created RedShift cluster using SQL WorkbenchJ

### Create the ‘accesslogs’ table in RedShift

    CREATE TABLE accesslogs
    (
            host_address varchar(512),
            request_time timestamp,
            request_method varchar(5),
            request_path varchar(1024),
            request_protocol varchar(10),
            response_code Int,
            response_size Int,
            referrer_host varchar(1024),
            user_agent varchar(512)
    )
    DISTKEY(host_address)
    SORTKEY(request_time);
    

### Loading data from S3 to Redshift

    COPY accesslogs 
    FROM 's3://YOUR-S3-BUCKET/access-log-processed' 
    CREDENTIALS 'aws_access_key_id=YOUR-ACCESS-KEY;aws_secret_access_key=YOUR-SECRET-KEY'
    DELIMITER '\t'
    MAXERROR 0
    GZIP;
    

### Execute these queries and on RedShift

    SELECT TRUNC(request_time), response_code, COUNT(1) FROM accesslogs GROUP BY 1,2 ORDER BY 1,3 DESC;
    

\*

    SELECT COUNT(1) FROM accessLogs WHERE response_code = 404;
    

\*

    SELECT TOP 1 request_path,COUNT(1) FROM accesslogs WHERE response_code = 404 GROUP BY 1 ORDER BY 2 DESC;
    

### Work with QuickSight

*   You can request access to QuickSight [here](https://aws.amazon.com/quicksight/)
*   Alternatively you can use any open source visualization tool that supports jdbc / odbc connections to databases


Efficient Data Lake Management with Apache Hudi Cleaner: Benefits of Scheduling Data Cleaning

![12312](https://user-images.githubusercontent.com/39345855/230497227-ed9e4729-a714-44e9-a89a-73ed0794c69a.JPG)


PDF  Guide how to setup Hudi in Glue 4.0
https://drive.google.com/file/d/1mkED3AUZBARsgeRCzk0K0XMvSyajo7mQ/view


PPT 
* https://docs.google.com/presentation/d/1QzSSoASG63qwFglU5aX7GtpCYDCd24pZ/edit?usp=sharing&ouid=110372297813046523203&rtpof=true&sd=true

Playlist 
* https://www.youtube.com/watch?v=CEzgFtmVjx4&list=PLL2hlSFBmWwy6RrazkVLni_U6xe1QdbqC

# lambda Functions 
```
try:
    import json
    import uuid
    import os
    import boto3

    from dotenv import load_dotenv
    load_dotenv("../.env")
except Exception as e:
    pass

global AWS_ACCESS_KEY
global AWS_SECRET_KEY
global AWS_REGION_NAME

AWS_ACCESS_KEY = os.getenv("DEV_ACCESS_KEY")
AWS_SECRET_KEY = os.getenv("DEV_SECRET_KEY")
AWS_REGION_NAME = "us-east-1"

client = boto3.client("emr-serverless",
                      aws_access_key_id=AWS_ACCESS_KEY,
                      aws_secret_access_key=AWS_SECRET_KEY,
                      region_name=AWS_REGION_NAME)


def lambda_handler_emr_job(event, context):
    # ------------------Hudi settings ---------------------------------------------
    path = os.getenv("TABLE_PATH")

    # ---------------------------------------------------------------------------------
    #                                       EMR
    # --------------------------------------------------------------------------------
    ApplicationId = os.getenv("ApplicationId")
    ExecutionTime = 600
    ExecutionArn = os.getenv("ExecutionArn")
    JobName = 'hudi_cleaner_job'

    # --------------------------------------------------------------------------------

    spark_submit_parameters = ' --conf spark.jars=/usr/lib/hudi/hudi-utilities-bundle.jar'
    spark_submit_parameters += ' --class org.apache.hudi.utilities.HoodieCleaner'

    arguments = [
        "--target-base-path", path,
        "--hoodie-conf", "hoodie.cleaner.policy=KEEP_LATEST_FILE_VERSIONS",
        "--hoodie-conf", "hoodie.cleaner.fileversions.retained=3",
        "--hoodie-conf", "hoodie.cleaner.parallelism=200",
        "--hoodie-conf", "hoodie.cleaner.commits.retained=5",

    ]

    response = client.start_job_run(
        applicationId=ApplicationId,
        clientToken=uuid.uuid4().__str__(),
        executionRoleArn=ExecutionArn,
        jobDriver={
            'sparkSubmit': {
                'entryPoint': "command-runner.jar",
                'entryPointArguments': arguments,
                'sparkSubmitParameters': spark_submit_parameters
            },
        },
        executionTimeoutMinutes=ExecutionTime,
        name=JobName,
    )
    print("response", end="\n")
    print(response)

lambda_handler_emr_job(event=None, context=None)
```


# Glue Job
```
"""
--conf spark.serializer=org.apache.spark.serializer.KryoSerializer --conf spark.sql.hive.convertMetastoreParquet=false --conf spark.sql.hive.convertMetastoreParquet=false --conf spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog --conf spark.sql.legacy.pathOptionBehavior.enabled=true

"""
try:
    import sys
    import os
    from pyspark.context import SparkContext
    from pyspark.sql.session import SparkSession
    from awsglue.context import GlueContext
    from awsglue.job import Job
    from awsglue.dynamicframe import DynamicFrame
    from pyspark.sql.functions import col, to_timestamp, monotonically_increasing_id, to_date, when
    from pyspark.sql.functions import *
    from awsglue.utils import getResolvedOptions
    from pyspark.sql.types import *
    from datetime import datetime, date
    import boto3
    from functools import reduce
    from pyspark.sql import Row

    import uuid
    from faker import Faker
except Exception as e:
    print("Modules are missing : {} ".format(e))

spark = (SparkSession.builder.config('spark.serializer', 'org.apache.spark.serializer.KryoSerializer') \
         .config('spark.sql.hive.convertMetastoreParquet', 'false') \
         .config('spark.sql.catalog.spark_catalog', 'org.apache.spark.sql.hudi.catalog.HoodieCatalog') \
         .config('spark.sql.extensions', 'org.apache.spark.sql.hudi.HoodieSparkSessionExtension') \
         .config('spark.sql.legacy.pathOptionBehavior.enabled', 'true').getOrCreate())

sc = spark.sparkContext
glueContext = GlueContext(sc)
job = Job(glueContext)
logger = glueContext.get_logger()

# =================================INSERTING DATA =====================================
global faker
faker = Faker()


class DataGenerator(object):

    @staticmethod
    def get_data():
        return [
            (
                uuid.uuid4().__str__(),
                faker.name(),
                faker.random_element(elements=('IT', 'HR', 'Sales', 'Marketing')),
                faker.random_element(elements=('CA', 'NY', 'TX', 'FL', 'IL', 'RJ')),
                str(faker.random_int(min=10000, max=150000)),
                str(faker.random_int(min=18, max=60)),
                str(faker.random_int(min=0, max=100000)),
                str(faker.unix_time()),
                faker.email(),
                faker.credit_card_number(card_type='amex'),

            ) for x in range(100)
        ]


# ============================== Settings =======================================
db_name = "hudidb"
table_name = "employees"
recordkey = 'emp_id'
precombine = "ts"
PARTITION_FIELD = 'state'
path = "s3://XXXXXXXXXXX/hudi/"
method = 'upsert'
table_type = "MERGE_ON_READ"
# ====================================================================================

hudi_part_write_config = {
    'className': 'org.apache.hudi',

    'hoodie.table.name': table_name,
    'hoodie.datasource.write.table.type': table_type,
    'hoodie.datasource.write.operation': method,
    'hoodie.datasource.write.recordkey.field': recordkey,
    'hoodie.datasource.write.precombine.field': precombine,
    "hoodie.schema.on.read.enable": "true",
    "hoodie.datasource.write.reconcile.schema": "true",

    'hoodie.datasource.hive_sync.mode': 'hms',
    'hoodie.datasource.hive_sync.enable': 'true',
    'hoodie.datasource.hive_sync.use_jdbc': 'false',
    'hoodie.datasource.hive_sync.support_timestamp': 'false',
    'hoodie.datasource.hive_sync.database': db_name,
    'hoodie.datasource.hive_sync.table': table_name,

    "hoodie.clean.automatic": "false"
    , "hoodie.clean.async": "false"
}

for i in range(1, 4):
    data = DataGenerator.get_data()
    columns = ["emp_id", "employee_name", "department", "state", "salary", "age", "bonus", "ts", "email", "credit_card"]
    spark_df = spark.createDataFrame(data=data, schema=columns)
    spark_df.write.format("hudi").options(**hudi_part_write_config).mode("append").save(path)

```

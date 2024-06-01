# Weather-Data-Analysis

## Overview
This project aims to automate the ingestion and processing of weather data using various AWS services. The workflow is designed to extract weather data from the OpenWeather API, store it in Amazon S3, process it using AWS Glue, and finally load the processed data into Amazon Redshift for further analysis. The project uses Amazon MWAA (Managed Workflows for Apache Airflow) to orchestrate the entire process.

## Project Structure
```
.
├── README.md  
├── Result_data  
│ └── weather_api_data.csv  
├── architecture.png  
├── buildspec.yml  
├── dags  
│ ├── openweather_api.py  
│ └── transform_redshift_load.py  
├── requirements.txt  
└── scripts  
└── weather_data_ingestion.py
```


## Architecture
![Architecture](./architecture.png)

## AWS Services Used
- **Amazon S3**: Acts as a data lake to store raw weather data.
- **AWS Glue**: Performs ETL (Extract, Transform, Load) operations and loads data into Redshift.
- **Amazon Redshift**: Data warehousing solution to store and analyze processed data.
- **Amazon SNS**: Sends notifications on the status of data processing.
- **Amazon MWAA**: Orchestrates the workflow using Apache Airflow.

## Workflow
1. **API Data Extraction**: 
   - A DAG in Airflow calls the OpenWeather API and ingests the data in CSV format into an S3 bucket.
2. **Data Processing and Load**: 
   - Another DAG triggers a Glue job which processes the ingested data and loads it into Amazon Redshift.
3. **Glue Job**:
   - The Glue job filters the weather data and transforms it before loading it into Redshift.

## Setup Instructions

### Prerequisites
- AWS account with appropriate permissions.
- OpenWeather API key.

### Step-by-Step Guide

1. **Clone the Repository**
    ```bash
    git clone <your-repository-url>
    cd <your-repository-directory>
    ```

2. **Create a Development Branch**
    ```bash
    git checkout -b dev
    ```

3. **Add Your Files to the Branch and Push**
    ```bash
    git add .
    git commit -m "Initial commit"
    git push origin dev
    ```

4. **Create a Pull Request and Merge**
    - Create a pull request from `dev` to `main` branch.
    - Once the pull request is merged, it will trigger the CI/CD pipeline using AWS CodeBuild.

5. **AWS CodeBuild**
    - CodeBuild will deploy your DAGs and scripts to the appropriate S3 buckets.
    - Ensure that the `buildspec.yml` file is correctly configured.

6. **Create MWAA Environment**
    - Create an MWAA environment using the Airflow bucket and `requirements.txt`.

### Airflow DAGs

#### `openweather_api_dag`
Extracts weather data from the OpenWeather API and uploads it to S3.

```python
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from airflow.models import Variable
from datetime import datetime, timedelta
import requests
import pandas as pd

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2023, 8, 12),
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG('openweather_api_dag', default_args=default_args, schedule_interval="@once", catchup=False)

api_endpoint = "https://api.openweathermap.org/data/2.5/forecast"
api_params = {
    "q": "Toronto,Canada",
    "appid": Variable.get("api_key")
}

def extract_openweather_data(**kwargs):
    response = requests.get
```
#### `transform_redshift_dag`
Creates and runs a Glue job to load data into Redshift.

```python
from airflow import DAG
from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
from airflow.sensors.external_task_sensor import ExternalTaskSensor
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2023, 8, 12),
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG('transform_redshift_dag', default_args=default_args, schedule_interval="@once", catchup=False)

transform_task = GlueJobOperator(
    task_id='transform_task',
    job_name='glue_transform_task',
    script_location='s3://aws-glue-assets-484478793255-us-east-1/scripts/weather_data_ingestion.py',
    s3_bucket='s3://aws-glue-assets-484478793255-us-east-1',
    aws_conn_id='aws_default',
    region_name="us-east-1",
    iam_role_name='airline-glue-role',
    create_job_kwargs={
        "GlueVersion": "4.0", 
        "NumberOfWorkers": 2, 
        "WorkerType": "G.1X",
        "Connections": {"Connections": ["Weather Redshift connection"]}
    },
    dag=dag,
)
```

### Glue Job Script
The Glue job script `weather_data_ingestion.py` performs data transformation and loads data into Redshift.

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
import datetime
  
args = getResolvedOptions(sys.argv, ["JOB_NAME"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

current_date = datetime.datetime.now().strftime("%Y-%m-%d")

weather_dyf = glueContext.create_dynamic_frame.from_options(
    format_options={"quoteChar": '"', "withHeader": True, "separator": ","},
    connection_type="s3",
    format="csv",
    connection_options={
        "paths": [f"s3://weather-data-landingzone-nm/date={current_date}/weather_api_data.csv"],
        "recurse": True,
    },
    transformation_ctx="weather_dyf",
)

changeschema_weather_dyf = ApplyMapping.apply(
    frame=weather_dyf,
    mappings=[
        ("dt", "string", "dt", "string"),
        ("weather", "string", "weather", "string"),
        ("visibility", "string", "visibility", "string"),
        ("`main.temp`", "string", "temp", "string"),
        ("`main.feels_like`", "string", "feels_like", "string"),
        ("`main.temp_min`", "string", "min_temp", "string"),
        ("`main.temp_max`", "string", "max_temp", "string"),
        ("`main.pressure`", "string", "pressure", "string"),
        ("`main.sea_level`", "string", "sea_level", "string"),
        ("`main.grnd_level`", "string", "ground_level", "string"),
        ("`main.humidity`", "string", "humidity", "string"),
        ("`wind.speed`", "string", "wind", "string"),
    ],
    transformation_ctx="changeschema_weather_dyf",
)

redshift_output = glueContext.write_dynamic_frame.from_options(
    frame=changeschema_weather_dyf,
    connection_type="redshift",
    connection_options={
        "redshiftTmpDir": "s3://aws-glue-assets-484478793255-us-east-1/temporary/",
        "useConnectionProperties": "true",
        "aws_iam_role": "arn:aws:iam::484478793255:role/service-role/AmazonRedshift-CommandsAccessRole-20240527T194709",
        "dbtable": "public.weather_data",
        "connectionName": "Weather Redshift connection",
        "preactions": "DROP TABLE IF EXISTS public.weather_data; CREATE TABLE IF NOT EXISTS public.weather_data (dt VARCHAR, weather VARCHAR, visibility VARCHAR, temp VARCHAR, feels_like VARCHAR, min_temp VARCHAR, max_temp VARCHAR, pressure VARCHAR, sea_level VARCHAR, ground_level VARCHAR, humidity VARCHAR, wind VARCHAR);",
    },
    transformation_ctx="redshift_output",
)

job.commit()
```
### CI/CD with CodeBuild
The `buildspec.yml` file defines the build and deployment steps.
```python
version: 0.2
phases:
  pre_build:
    commands:
      - echo "Starting build process..."
  build:
    commands:
      - echo "Copying DAG files to S3..."
      - aws s3 cp --recursive ./dags s3://aws-airflow-nm/dags/
      - echo "Copying requirements.txt files to S3..."
      - aws s3 cp ./requirements.txt s3://aws-airflow-nm/
      - echo "Copying Glue scripts to S3..."
      - aws s3 cp --recursive ./scripts s3://aws-glue-assets-484478793255-us-east-1/scripts/
  post_build:
    commands:
      - echo "Build and deployment process complete!!!"
```

### How to Run
#### CI/CD Deployment of the project:
Commit and push your code chages to `dev` branch, then create a pull request to `main` branch.
Once you approve the pull request it triggers your codeBuild project.
#### CodeBuild deploys the Airflow DAGs and Glue Scripts:
Ensure the files are deployed to the correct S3 buckets using CodeBuild.
#### Trigger the Dags:
Use the Airflow UI to trigger the openweather_api_dag which will start the data ingestion process.
#### Monitor and Validate:
Monitor the status of the DAGs and Glue jobs in the Airflow and AWS Glue consoles.
Validate the data in the Redshift tables.

### Conclusion
This project demonstrates a scalable and automated workflow for ingesting and processing weather data using AWS services. By leveraging Airflow for orchestration, Glue for ETL, and Redshift for data warehousing, the pipeline ensures efficient data processing and analysis.

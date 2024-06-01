# Weather-Data-Analysis

## Overview
This project aims to automate the ingestion and processing of weather data using various AWS services. The workflow is designed to extract weather data from the OpenWeather API, store it in Amazon S3, process it using AWS Glue, and finally load the processed data into Amazon Redshift for further analysis. The project uses Amazon MWAA (Managed Workflows for Apache Airflow) to orchestrate the entire process.

## Project Structure
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


## Architecture
![Architecture](./architecture.png)

## AWS Services Used
- **Amazon S3**: Acts as a data lake to store raw weather data.
- **Amazon EventBridge**: Captures events and triggers workflows.
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

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

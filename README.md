# GCP Marketing Data Pipeline: API to BigQuery & Salesforce Integration

![GCP Status](https://img.shields.io/badge/Google_Cloud-Active-4285F4?style=for-the-badge&logo=google-cloud&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.9+-3776AB?style=for-the-badge&logo=python&logoColor=white)
![BigQuery](https://img.shields.io/badge/BigQuery-Data_Warehouse-669DF6?style=for-the-badge&logo=google-cloud&logoColor=white)

## 📋 Project Overview
This project consists of a serverless ETL pipeline designed to extract marketing data from the **Gaio Social API**, process it using **Python**, store it in **Google BigQuery**, and sync qualified leads to **Salesforce Marketing Cloud**.

The solution was engineered to solve a data fragmentation issue, creating a **Single Source of Truth (SSOT)** for the marketing and sales teams, enabling real-time attribution modeling.

### 🏗 Architecture
**Source (REST API) -> Cloud Functions (Python) -> BigQuery (Data Warehouse) -> Salesforce (Activation)**

1.  **Ingestion:** Scheduled Cloud Functions trigger Python scripts to fetch daily delta data.
2.  **Processing:** Data normalization, type casting, and PII handling using Pandas.
3.  **Storage:** Partitioned tables in BigQuery for cost-optimized querying.
4.  **Reverse ETL:** SQL-based segmentation pushes high-score leads to Salesforce via API.

## 🛠 Tech Stack
* **Cloud:** Google Cloud Platform (GCP)
* **Compute:** Cloud Run / Cloud Functions
* **Orchestration:** Cloud Scheduler
* **Data Warehouse:** Google BigQuery
* **Language:** Python 3.9+
* **Key Libraries:** `pandas`, `requests`, `google-cloud-bigquery`, `simple-salesforce`

## 🚀 Key Features
* **Automated Pagination Handling:** Python script robustly handles API pagination and rate limits.
* **Idempotency:** The pipeline is designed to be idempotent; re-running it does not duplicate data (uses `MERGE` statements).
* **Cost Optimization:** Implements `WRITE_TRUNCATE` for staging tables and partitioned historical tables to minimize storage costs.
* **Error Handling:** Integrated logging with Cloud Logging for rapid troubleshooting.

## 💻 Code Snippet (Core Extraction Logic)

```python
import requests
import pandas as pd
from google.cloud import bigquery
import os

def fetch_marketing_data(api_endpoint, headers, params):
    """
    Fetches data from the marketing API with error handling and pagination.
    """
    all_data = []
    try:
        while api_endpoint:
            response = requests.get(api_endpoint, headers=headers, params=params)
            response.raise_for_status()
            data = response.json()
            all_data.extend(data.get('results', []))
            
            # Handle Pagination (Example)
            api_endpoint = data.get('next') 
            
        return pd.DataFrame(all_data)
        
    except requests.exceptions.RequestException as e:
        print(f"CRITICAL: API Connection failed: {e}")
        return None

def load_to_bigquery(dataframe, table_id):
    """
    Loads the processed dataframe into BigQuery using the Client Library.
    """
    client = bigquery.Client()
    job_config = bigquery.LoadJobConfig(
        write_disposition="WRITE_APPEND", # or WRITE_TRUNCATE for staging
        schema_update_options=[bigquery.SchemaUpdateOption.ALLOW_FIELD_ADDITION]
    )
    
    job = client.load_table_from_dataframe(
        dataframe, table_id, job_config=job_config
    )
    job.result()  # Wait for the job to complete
    
    print(f"SUCCESS: Loaded {job.output_rows} rows into {table_id}.")

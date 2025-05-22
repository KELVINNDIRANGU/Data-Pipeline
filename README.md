## Crypto ETL Pipeline Using Apache Airflow

This project demonstrates how to build an ETL (Extract, Transform, Load) data pipeline using [Apache Airflow](https://airflow.apache.org/) to fetch cryptocurrency data from [CoinGecko](https://www.coingecko.com/), transform it, and load it into a **PostgreSQL** database. The pipeline runs on an hourly schedule and stores prices of Bitcoin and Ethereum.

## Installation & Setup Guide (Ubuntu + Airflow)

## Step 1: Create and Activate Virtual Environment
sudo apt install python3.12-venv
python3 -m venv airflow
source airflow/bin/activate

## Step 2: Install Apache Airflow
pip install apache-airflow==2.9.1

## Step 3: Install PostgreSQL Dependencies
pip install psycopg2-binary

## DAG Overview
# dags/crypto_etl.py

from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime
import requests
import psycopg2

# PostgreSQL credentials - Replace with your actual credentials
DB_NAME = "airflow_db"
DB_USER = "peter"
DB_PASSWORD = "1234"
DB_HOST = "194.180.176.173"
DB_PORT = "5432"

# Extract: Pull cryptocurrency price data from CoinGecko API
def extract():
    url = "https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd"
    response = requests.get(url)
    return response.json()

# Transform: Clean and reshape the JSON response into a list of tuples
def transform(raw_data):
    data = []
    for coin, price in raw_data.items():
        data.append((coin, price['usd'], datetime.now()))
    return data

# Load: Insert the cleaned data into a PostgreSQL table
def load(data):
    conn = psycopg2.connect(
        dbname=DB_NAME,
        user=DB_USER,
        password=DB_PASSWORD,
        host=DB_HOST,
        port=DB_PORT
    )

    cur = conn.cursor()

    # Create schema and table if not already existing
    cur.execute("""
        CREATE SCHEMA IF NOT EXISTS kelvin;
        CREATE TABLE IF NOT EXISTS kelvin.crypto_prices (
            coin TEXT,
            usd_price NUMERIC,
            timestamp TIMESTAMP
        );
    """)

    # Insert data into the table
    cur.executemany(
        "INSERT INTO kelvin.crypto_prices (coin, usd_price, timestamp) VALUES (%s, %s, %s);",
        data
    )

    conn.commit()
    cur.close()
    conn.close()

# Combined ETL pipeline
def etl():
    raw = extract()
    clean = transform(raw)
    load(clean)

# Define DAG and schedule
with DAG(
    dag_id="crypto_etl",
    description = "ETL pipeline to fetch Bitcoin and Ethereum prices and store them in PostgreSQL",
    start_date=datetime(2025, 1, 1),
    schedule_interval="@hourly",
    catchup=False,
    tags=["crypto", "ETL", "postgres", "api"]
) as dag:

    run_etl = PythonOperator(
        task_id="run_etl_task",
        python_callable=etl
    )

    run_etl

## Running the Pipeline

## Step 1: Start Airflow Webserver
airflow webserver

## Step 2: Start the Scheduler
airflow scheduler

## Step 3: Access the Airflow UI

Open your browser and go to:
http://localhost:8080
Log in and trigger the `crypto_etl` DAG manually or wait for its hourly schedule.

## Features
Uses Python + Airflow to automate data pipelines
Fetches live API data (Bitcoin & Ethereum)
Stores data in PostgreSQL (schema: `kelvin.crypto_prices`)
Modular ETL structure
Easily extensible for more coins or different transformations

## Sample Query for Analysis

SELECT coin, AVG(usd_price) AS avg_price
FROM kelvin.crypto_prices
GROUP BY coin;



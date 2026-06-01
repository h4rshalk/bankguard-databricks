# bankguard-databricks
# BankGuard — Automated Fraud Detection Pipeline on Databricks

![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=flat&logo=databricks&logoColor=white)
![PySpark](https://img.shields.io/badge/PySpark-E25A1C?style=flat&logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-003366?style=flat)
![MLflow](https://img.shields.io/badge/MLflow-0194E2?style=flat&logo=mlflow&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-4479A1?style=flat&logo=mysql&logoColor=white)

An end-to-end automated fraud detection pipeline built on Databricks Community Edition — simulating how real fintech and banking companies process transaction data daily using industry-standard architecture and tooling.

---

## Problem Statement

Banks and fintech companies process millions of transactions daily. Fraudulent transactions often go undetected for days — by the time a customer reports fraud, the damage is done. This project builds an automated pipeline that ingests daily transaction data, cleans and enriches it, aggregates fraud intelligence metrics, and trains a machine learning model to flag suspicious transactions — all running automatically every day without manual intervention.

---

## Architecture

```
Raw Transactions (Python Generator)
           │
           ▼
┌─────────────────────┐
│   BRONZE LAYER      │  ← Raw data ingested as-is into Delta table
│  bronze_transactions│    No modifications — permanent audit trail
└─────────┬───────────┘
           │
           ▼
┌─────────────────────┐
│   SILVER LAYER      │  ← Cleaned, validated, enriched data
│  silver_transactions│    PySpark transformations + feature engineering
└─────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────────┐
│              GOLD LAYER                      │  ← Business-ready aggregations
│  gold_daily_summary   gold_fraud_by_city     │
│  gold_fraud_by_type   gold_high_risk_customers│
└──────────────────────┬───────────────────────┘
                       │
           ┌───────────┴───────────┐
           ▼                       ▼
┌─────────────────┐     ┌──────────────────────┐
│   ML MODEL      │     │   DAILY REPORT       │
│ XGBoost +       │     │ Automated fraud       │
│ MLflow Tracking │     │ intelligence summary  │
└─────────────────┘     └──────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│         DATABRICKS WORKFLOWS                │
│  Automated daily pipeline — 7:00 AM         │
│  bronze → silver → gold → report            │
│  Dependency managed, failure alerts enabled │
└─────────────────────────────────────────────┘
```

This follows the **Medallion Architecture** — the industry standard pattern used by data engineering teams at companies like Uber, Airbnb, Razorpay, and PhonePe.

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Data Processing | Apache Spark / PySpark | Distributed data transformation |
| Storage | Delta Lake | ACID transactions, time travel, schema enforcement |
| Platform | Databricks Community Edition | Unified data + ML environment |
| ML Tracking | MLflow | Experiment tracking, model registry |
| ML Model | XGBoost | Fraud classification |
| Automation | Databricks Workflows | Daily pipeline scheduling |
| Language | Python, SQL | Transformations and aggregations |

---

## Project Structure

```
bankguard-databricks/
│
├── 01_data_generation.ipynb     # Generates 10,000 realistic Indian banking transactions
│                                # Saves to bronze_transactions Delta table
│
├── 02_silver_layer.ipynb        # PySpark cleaning and feature engineering
│                                # Removes bad data, adds is_late_night, amount_category
│                                # Saves to silver_transactions Delta table
│
├── 03_gold_layer.ipynb          # Business aggregations using Spark SQL
│                                # Creates 4 Gold Delta tables for analytics
│
├── 04_daily_report.ipynb        # Automated fraud intelligence report
│                                # Reads from Gold tables, prints daily summary
│
└── README.md
```

---

## Dataset

Synthetically generated Indian banking transaction data with realistic fraud patterns:

| Field | Description |
|---|---|
| transaction_id | Unique transaction identifier |
| customer_id | Customer identifier (500 unique customers) |
| amount | Transaction amount (exponential distribution, realistic spread) |
| city | Indian city (Mumbai, Delhi, Bangalore, Hyderabad, etc.) |
| transaction_type | UPI, NEFT, IMPS, Credit Card, Debit Card |
| merchant | Amazon, Flipkart, Swiggy, Zomato, Uber, etc. |
| transaction_date | Date and time of transaction |
| hour_of_day | Hour extracted for late-night fraud detection |
| day_of_week | Day extracted for weekend pattern analysis |
| is_fraud | Target variable (1 = fraud, 0 = legitimate) |

**Fraud logic:** Transactions are marked fraud based on realistic patterns — high amounts (>₹15,000), late night timing (11pm–5am), and IMPS transaction type have higher fraud probability. Overall fraud rate: ~1%, matching real-world banking data.

---

## What Each Layer Does

### Bronze Layer
Ingests raw transaction data exactly as received — no modifications. Acts as the permanent audit trail. If anything goes wrong in downstream layers, data engineers always have the original copy to recover from.

### Silver Layer
Applies PySpark transformations to clean and enrich the data:
- Removes negative/zero amount transactions
- Drops duplicate transaction IDs
- Enforces correct data types
- Adds `is_weekend` — flags Saturday/Sunday transactions
- Adds `is_late_night` — flags transactions between 11pm and 5am
- Adds `amount_category` — Low / Medium / High / Very High
- Adds `processed_at` timestamp for pipeline auditing

### Gold Layer
Creates 4 business-ready aggregation tables:

**gold_daily_summary** — Total transactions, total amount, fraud count, fraud rate per day

**gold_fraud_by_city** — Fraud rate ranked by city (Hyderabad highest at 1.36%, Delhi lowest at 0.59%)

**gold_fraud_by_type** — Fraud rate by transaction type (IMPS highest at 1.52%)

**gold_high_risk_customers** — Top 50 customers by fraud transaction count with total spend

### ML Model
- Algorithm: XGBoost Classifier
- Features: amount, hour_of_day, is_weekend, is_late_night, transaction_type, city, merchant
- Class imbalance handled with `scale_pos_weight=10`
- All runs tracked with MLflow — parameters, AUC, precision, recall, F1, model artifact
- Training / test split: 80/20 with stratification

### Databricks Workflows
Automated pipeline with 4 dependent tasks:
```
bronze_data_generation → silver_layer → gold_layer → daily_report
```
Scheduled daily at 7:00 AM on Serverless compute. Each task only runs if the previous one succeeded.

---

## Key Results

| Metric | Value |
|---|---|
| Total transactions processed | 10,000 |
| Total transaction value | ₹2.93 Crore |
| Fraud transactions detected | 106 |
| Overall fraud rate | 1.06% |
| Highest risk city | Hyderabad (1.36%) |
| Highest risk transaction type | IMPS (1.52%) |
| ML Model AUC | 0.55 |
| Pipeline run duration | ~4 minutes end to end |

**Note on ML metrics:** The AUC of 0.55 is expected given only 106 fraud samples in training data. In production, this would be addressed with SMOTE oversampling, larger historical datasets, and threshold tuning — standard practice for imbalanced fraud detection problems.

---

## How to Run This Project

### Prerequisites
- Databricks Community Edition account (free at community.cloud.databricks.com)
- No additional cloud account or credit card required

### Steps

1. **Clone this repository** or download the notebooks

2. **Upload notebooks to Databricks**
   - Log in to Databricks Community Edition
   - Go to Workspace → Home
   - Create a folder called `bankguard`
   - Upload all 4 `.ipynb` files

3. **Run in order**
   - Open `01_data_generation.ipynb` → Run all cells
   - Open `02_silver_layer.ipynb` → Run all cells
   - Open `03_gold_layer.ipynb` → Run all cells
   - Open `04_daily_report.ipynb` → Run all cells

4. **Set up automation (optional)**
   - Go to Jobs & Pipelines → Create Job
   - Add all 4 notebooks as tasks in order
   - Set dependencies: each task depends on the previous
   - Schedule at your preferred time

5. **Verify tables in Catalog**
   - Go to Catalog in left sidebar
   - You should see 6 Delta tables: bronze_transactions, silver_transactions, and 4 gold tables

---

## Concepts Demonstrated

- **Medallion Architecture** (Bronze → Silver → Gold)
- **Delta Lake** — ACID transactions, schema enforcement, time travel
- **PySpark** — DataFrame transformations, feature engineering, aggregations
- **MLflow** — Experiment tracking, parameter logging, model registry
- **Databricks Workflows** — Pipeline automation, dependency management, scheduling
- **Databricks Catalog** — Delta table governance and discovery
- **Fraud detection** — Class imbalance handling, business-relevant feature engineering

---

## Author

**Harshal Kawane**
Data Analyst | Databricks · PySpark · Power BI · SQL · Python

[LinkedIn](https://www.linkedin.com/in/harshal-kawane/) · [GitHub](https://github.com/h4rshalk) · [Kaggle](https://www.kaggle.com/h4rshal3)

---

*Built on Databricks Community Edition — free tier. No cloud account or paid subscription required to run this project.*

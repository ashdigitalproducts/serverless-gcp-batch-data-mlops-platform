# 📦 Outline of Code Files for the Serverless Batch Platform

This document outlines the purpose and logic of the key code files that power the `winter-days` project.

**NOTE:** This document does not contain the full source code, only a high-level description of each component's logic.

---

## <br> 🚀 1. Data Generators (Cloud Run Jobs)
Three separate, containerized Python jobs are responsible for simulating and ingesting data for the three flows.

### <br> 1.1. `winter-days-firestore-sales-gen/main.py`
* **Purpose:** Generates `v2.0.0` sales data and writes it to the **Cloud Firestore** source database.
* **Logic:**
    * Generates a random number of records (5-20).
    * Creates a rich record with new fields (`status`, `payment_method`, `contact_email_hashed`, `credit_card_last4`, etc.).
    * Saves the `sale_timestamp` as an **ISO-formatted string**.
    * Uses the Firestore client library to write these records to the `winter_sales` collection.
    * Runs as the `winter-days-firestore-sales-gen-sa` service account.

### <br> 1.2. `winter-days-clickstream-gen/main.py`
* **Purpose:** Generates `v2.0.0` clickstream data and writes it directly to the **GCS Bronze Layer**.
* **Logic:**
    * Generates 1000 JSONL records.
    * Enriches the data with new fields like `referrer_domain`, `device_type_category`, `is_bounce`, and `load_timestamp`.
    * Uploads the final `.jsonl` file to the `gs://.../clickstream/` folder.
    * Runs as the `winter-days-clickstream-gen-sa` service account.

### <br> 1.3. `winter-days-startup-gen/main.py`
* **Purpose:** Generates `v2.0.0` startup data and writes it directly to the **GCS Bronze Layer**.
* **Logic:**
    * Generates 100 JSONL records.
    * Enriches the data with new fields like `funding_stage_score`, `valuation_bucket`, `contact_email_cleaned` (hashed), and `is_active`.
    * Uploads the final `.jsonl` file to the `gs://.../startups/` folder.
    * Runs as the `winter-days-startup-gen-sa` service account.

---

## <br> ⚙️ 2. Master Orchestrator (Cloud Run Service)
This is the "brain" of the entire platform, packaged as a single `server.py` file.

### <br> 2.1. `winter-days-orchestrator-code/server.py`
* **Purpose:** A Flask web service that, when triggered, runs the entire end-to-end MLOps pipeline.
* **Trigger:** Called by a Cloud Scheduler job via an authenticated HTTP POST request.
* **Identity:** Runs as the `winter-days-orchestrator-sa` service account.
* **Step-by-Step Logic:**
    1.  **`export_sales_to_gcs()`:** Connects to Firestore, pulls all records from `winter_sales`, uses `pandas` to convert them to a DataFrame, and saves them as a Parquet file in the `gs://.../sales/` (Bronze) folder. Includes the robust `robust_timestamp_parser` to handle mixed `string` and `datetime` data.
    2.  **`create_compilation_result()`:** Makes an HTTP request to the Dataform API to compile the code in the `batch-workspace`.
    3.  **`trigger_dataform_pipeline()`:** Uses the ID from Step 2 to make a *second* HTTP request, triggering a new Dataform workflow. It correctly specifies the `dataform-exec-sa` as the service account for the job to run as.
    4.  **`poll_dataform_status()`:** Enters a loop (max 30 mins) and pings the Dataform API every 60 seconds, waiting for the workflow to return `SUCCEEDED` or `FAILED`.
    5.  **`trigger_mlops_pipeline()`:** If Dataform succeeds, this function uses the `google-cloud-aiplatform` SDK to submit the Vertex AI training pipeline, using the `customer_churn_pipeline.json` file as its template.
    6.  **Response:** The service returns a `200 OK` JSON payload if all steps (up to pipeline submission) succeed, or a `500` error if any step fails.

---

## <br> SQL 3. ELT Pipeline (Dataform SQLX Files)
This code lives *inside* the Dataform repository (`winter-days-data-repo`) and is described in `all current sqlx files.txt`.

* **`definitions/sales/ext_sales_gcs.sqlx`:** Creates the BigQuery External Table pointing to the `/sales/` Parquet files in the Bronze layer.
* **`definitions/sales/stg_sales.sqlx`:** Cleans, types, and pseudonymizes the sales data into the `winter_days_staging` dataset (Silver Layer).
* **`definitions/sales/agg_cust_metrics.sqlx`:** Creates the key customer-level feature table for the ML model (Gold Layer).
* **`definitions/sales/fct_sales.sqlx` & `dim_customer.sqlx`:** Builds the final, business-ready Warehouse tables (Gold Layer).
* **(Similar files exist for `clickstream` and `startup` flows):**
    * `ext_clickstream_gcs.sqlx` $\rightarrow$ `stg_click.sqlx` $\rightarrow$ `fct_click_features.sqlx`
    * `ext_startup_gcs.sqlx` $\rightarrow$ `stg_startup.sqlx` $\rightarrow$ `dim_startup.sqlx`
 
---

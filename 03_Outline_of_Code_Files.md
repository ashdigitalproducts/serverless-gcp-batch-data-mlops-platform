# 📦 Outline of Code Files for the Serverless Batch Platform

This document outlines the purpose and logic of the key code files that power this serverless project.

**NOTE:** This document does not contain the full source code, only a high-level description of each component's logic.

---

## <br> 🚀 1. Data Generators (Cloud Run Jobs)
Three separate, containerized Python jobs are responsible for simulating and ingesting data for the three flows.

### <br> 1.1. The Sales Data Generator  
* **Purpose:** Generates data and writes it to the **Cloud Firestore** source database.
* **Logic:**
    * Generates a random batch of records.
    * Creates a rich record with derived fields and applies hashing to PII fields.
    * Saves timestamps as **ISO-formatted strings** for consistent parsing.
    * Uses the Firestore client library to write records to the target sales collection.
    * Runs using a dedicated, least-privilege service account for writing to Firestore.

### <br> 1.2. The Clickstream Data Generator  
* **Purpose:** Generates data and writes it directly to the **GCS Bronze Layer**.
* **Logic:**
    * Generates a batch of JSONL records.
    * Enriches the data with new fields.
    * Uploads the final file to the folder in GCS.
    * Runs using a dedicated, least-privilege service account for writing to GCS.

### <br> 1.3. The Startup Data Generator  
* **Purpose:** Generates data and writes it directly to the **GCS Bronze Layer**.
* **Logic:**
    * Generates a batch of JSONL records.
    * Enriches the data with new fields.
    * Uploads the final file to the folder in GCS.
    * Runs using a dedicated, least-privilege service account for writing to GCS.

---

## <br> ⚙️ 2. Master Orchestrator (Cloud Run Service)
This is the "brain" of the entire platform, packaged as a single Python application.

### <br> 2.1. The Orchestrator Application  
* **Purpose:** A Flask web service that, when triggered, runs the entire end-to-end MLOps pipeline.
* **Trigger:** Called by a Cloud Scheduler job via an authenticated HTTP POST request.
* **Identity:** Runs as a dedicated, least-privilege Master Orchestrator service account.
* **Step-by-Step Logic:**
    1.  **Firestore to GCS Export:** Connects to Firestore, pulls all records from the collection, uses `pandas` and `pyarrow` to convert them to a Parquet file, and lands it in the Bronze folder. Includes a robust parser for mixed timestamp formats.
    2.  **Dataform Compilation:** Makes an HTTP request to the Dataform API to compile the code from a specified development workspace.
    3.  **Dataform Trigger:** Uses the compilation ID from Step 2 to make a *second* HTTP request, triggering a new Dataform workflow. It specifies a dedicated Dataform execution SA (as required by IAM "act as" security).
    4.  **Poll for Completion:** Enters a loop, pinging the Dataform API until the workflow returns `SUCCEEDED` or `FAILED`.
    5.  **Trigger MLOps:** If Dataform succeeds, this function uses the Vertex AI SDK to submit the training pipeline, referencing a compiled pipeline spec (`.json`) from GCS.
    6.  **Response:** The service returns a `200 OK` JSON payload if all steps (up to pipeline submission) succeed, or a `500` error if any step fails.

### <br> 2.2. Orchestrator Dependencies  
* **`Requirements File`:** Lists all necessary Python libraries (e.g., `flask`, `google-cloud-firestore`, `google-cloud-storage`, `google-cloud-aiplatform`, `pandas`, `pyarrow`, `requests`).
* **`Dockerfile`:** Defines the container image, installing all dependencies and setting the `gunicorn` web server as the entrypoint to run the Flask app.

---

## <br> SQL 3. ELT Pipeline (Dataform SQLX Files)
This code lives *inside* the Dataform repository and defines the entire Medallion Architecture in SQL for all the 3 data flows: 
    * Creates a BigQuery External Table pointing to the raw sales Parquet files in the Bronze layer.
    * Cleans, types, and pseudonymizes raw sales data from the external table into a Staging (Silver) table.
    * Aggregates customer-level features (e.g., `avg_order_value`, `total_revenue`) to create a table for ML model training.
    * Builds the final, business-ready Fact and Dimension tables for the BI dashboard.

---

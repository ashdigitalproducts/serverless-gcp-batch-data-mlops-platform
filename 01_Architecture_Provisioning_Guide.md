# 🏛️ Full Architecture Provisioning Guide
The following clearly describes all the GCP Services and Resources provisioned for this 100% Serverless Batch Data & MLOps Platform.

## <br> 🚀 1. Cloud Project Configuration
Setup the GCP Project and set the regional context to the target Region for all subsequent deployments.

---

## <br> 🚀 2. API Enablement
Enable all necessary APIs:
* **Orchestration & Compute:** Cloud Run API (`run.googleapis.com`), Cloud Build API (`cloudbuild.googleapis.com`)
* **Triggers:** Cloud Scheduler API (`cloudscheduler.googleapis.com`)
* **Storage & Database:** Cloud Storage API (`storage.googleapis.com`), Cloud Firestore API (`firestore.googleapis.com`)
* **Data Warehouse:** BigQuery API (`bigquery.googleapis.com`)
* **ELT Orchestration:** Dataform API (`dataform.googleapis.com`)
* **BQ External Connections:** BigQuery Connection API (`bigqueryconnection.googleapis.com`)
* **Unified AI Platform:** Vertex AI API (`aiplatform.googleapis.com`)
* **Container Registry:** Artifact Registry API (`artifactregistry.googleapis.com`)
* **Underlying Compute:** Compute Engine API (`compute.googleapis.com`)
* **Security:** IAM API (`iam.googleapis.com`)

---

## <br> 🚀 3. Service Accounts & IAM
This architecture runs on a "Least Privilege" model using multiple custom Service Accounts (SAs).

* **`winter-days-scheduler-sa`:**
    * **Purpose:** Triggers all automated jobs.
    * **Roles:** `roles/run.invoker` on all three Data Generator Cloud Run Jobs and the Master Orchestrator Cloud Run Service.

* **`winter-days-firestore-sales-gen-sa`:**
    * **Purpose:** Runs the Sales data generator job.
    * **Roles:** `roles/datastore.user` (to write to Firestore).

* **`winter-days-clickstream-gen-sa`:**
    * **Purpose:** Runs the Clickstream data generator job.
    * **Roles:** `roles/storage.objectCreator` (to write JSONL files to the GCS Bronze bucket).

* **`winter-days-startup-gen-sa`:**
    * **Purpose:** Runs the Startup data generator job.
    * **Roles:** `roles/storage.objectCreator` (to write JSONL files to the GCS Bronze bucket).

* **`winter-days-orchestrator-sa`:**
    * **Purpose:** The "Master Orchestrator." The most critical SA that runs the main `server.py` service.
    * **Roles:**
        * `roles/datastore.user` (to read from Firestore).
        * `roles/storage.objectCreator` (to write Parquet files to GCS Bronze).
        * `roles/dataform.editor` (to create compilations and trigger workflow invocations).
        * `roles/aiplatform.user` (to submit Vertex AI Pipeline jobs).
        * `roles/iam.serviceAccountUser` **(Conditional)**: A binding on the *project* with a condition restricting it *only* to the Google-managed Dataform SA. This is the crucial permission that allows the orchestrator to "act as" the Dataform SA.

* **`dataform-exec-sa` (User-Managed):**
    * **Purpose:** The identity that Dataform *uses* to run its SQL jobs. This is specified in the orchestrator's `server.py` file.
    * **Roles:**
        * `roles/bigquery.dataEditor` (to create/write to tables in `winter_days_staging` and `winter_days_warehouse`).
        * `roles/bigquery.jobUser` (to run BigQuery jobs).
        * `roles/storage.objectViewer` (to read the raw files from the GCS Bronze bucket).

* **`service-184590945141@gcp-sa-dataform.iam.gserviceaccount.com` (Google-Managed):**
    * **Purpose:** The default Google-managed SA for Dataform.
    * **Note:** We grant the `winter-days-orchestrator-sa` "act as" permission on this SA (via IAM condition) to allow it to trigger jobs.

* **`winter-days-vertex-ai-sa` (Planned):**
    * **Purpose:** The identity for the Vertex AI Pipeline to run as.
    * **Roles:** `roles/bigquery.dataViewer` (to read training data), `roles/bigquery.dataEditor` (to write predictions), `roles/storage.objectAdmin` (to manage pipeline artifacts).

---

## <br> 🚀 4. Core Data Infrastructure

* **Cloud Storage (GCS) Bucket:**
    * **Name:** `winter-days-nov-2025-lake-raw`
    * **Purpose:** The **Bronze Layer** for the Medallion Architecture.
    * **Folders:** `/sales/`, `/clickstream/`, `/startups/` hold the raw, immutable data files. Also holds MLOps pipeline artifacts in `/mlops/`.
* **Cloud Firestore Database:**
    * **Mode:** Native
    * **Collection:** `winter_sales`
    * **Purpose:** The NoSQL source database for the Sales data flow.
* **BigQuery Datasets:**
    * **`winter_days_staging`:** Holds all intermediate tables (`stg_sales`, `stg_click`, etc.) created by Dataform. This is the **Silver Layer**.
    * **`winter_days_warehouse`:** Holds the final, business-ready tables. This is the **Gold Layer**.
        * `fct_sales` (Fact Table)
        * `dim_customer` (Dimension Table)
        * `fct_click_features` (Fact Table)
        * `dim_startup` (Dimension Table)
        * `fact_predictions_churn` (ML Output Table)

---

## <br> 🚀 5. Batch Ingestion & Orchestration

* **Data Generators (Cloud Run Jobs):**
    1.  **`winter-days-firestore-sales-gen`:** Scheduled to run daily. Generates `v2.0.0` sales data and writes to Firestore.
    2.  **`winter-days-clickstream-gen`:** Scheduled to run daily. Generates `v2.0.0` clickstream JSONL and writes to GCS.
    3.  **`winter-days-startup-gen`:** Scheduled to run weekly. Generates `v2.0.0` startup JSONL and writes to GCS.
* **Master Orchestrator (Cloud Run Service):**
    * **Name:** `winter-days-batch-orchestrator`
    * **Source:** Built from `server.py`, `Dockerfile`, `requirements.txt`.
    * **Trigger:** A Cloud Scheduler job runs daily (e.g., at 2 AM) to call this service's HTTP endpoint.
    * **Logic:** Executes the full 5-step batch pipeline.

---

## <br> 🚀 6. ELT (Dataform)

* **Repository:** `winter-days-data-repo`
* **Workspace:** `batch-workspace` (This is the workspace the orchestrator compiles).
* **Logic:** A collection of `definitions/` with SQLX files that:
    1.  Create External Tables pointing to the GCS Bronze files (e.g., `ext_sales_gcs.sqlx`).
    2.  Clean, type, and pseudonymize data into `stg_` tables (Silver Layer).
    3.  Aggregate and model data into final `fct_` and `dim_` tables (Gold Layer).

---

## <br> 🚀 7. MLOps Layer (Planned)

* **ML Pipeline Spec:** `customer_churn_pipeline.json`
    * **Source:** A file stored in GCS (`gs://.../mlops/`) that defines the MLOps pipeline steps.
* **Trigger:** The Master Orchestrator calls `aiplatform.PipelineJob().submit()` after the Dataform polling succeeds, launching the training pipeline.
* **Output:** The pipeline (when built) will train a churn model and save its predictions to the `fact_predictions_churn` table in BigQuery.

---

## <br> 🚀 8. Business Intelligence Layer

* **Tool:** Looker Studio
* **Strategy:** A two-phase dashboard:
    1.  **Pre-MLOps:** Connects to `fct_sales`, `dim_customer`, etc., to show historical BI ("What happened?").
    2.  **Post-MLOps:** Adds `fact_predictions_churn` as a data source to show predictive insights ("What will happen?").
 
---

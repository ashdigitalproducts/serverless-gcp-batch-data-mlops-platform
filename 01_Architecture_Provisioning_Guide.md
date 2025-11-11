# 🏛️ Full Architecture Provisioning Guide
The following clearly describes all the GCP Services and Resources provisioned for this 100% Serverless Batch Data & MLOps Platform.

## <br> 🚀 1. Cloud Project Configuration
Setup the GCP Project and set the regional context to the target Region for all subsequent deployments.

---

## <br> 🚀 2. API Enablement
Enable all necessary APIs:
* **Orchestration & Compute:** Cloud Run API, Cloud Build API
* **Triggers:** Cloud Scheduler API
* **Storage & Database:** Cloud Storage API, Cloud Firestore API
* **Data Warehouse:** BigQuery API
* **ELT Orchestration:** Dataform API
* **BQ External Connections:** BigQuery Connection API
* **Unified AI Platform:** Vertex AI API
* **Container Registry:** Artifact Registry API
* **Underlying Compute:** Compute Engine API
* **Security:** IAM API

---

## <br> 🚀 3. Service Accounts & IAM
This architecture runs on a "Least Privilege" model using multiple custom Service Accounts (SAs).

* **A custom SA for Cloud Scheduler:**
    * **Purpose:** Triggers all automated jobs.
    * **Roles:** `roles/run.invoker` on all Data Generator Cloud Run Jobs and the Master Orchestrator Cloud Run Service.

* **A custom SA for the Sales Generator:**
    * **Purpose:** Runs the Sales data generator job.
    * **Roles:** `roles/datastore.user` (to write to Firestore).

* **A custom SA for the Clickstream Generator:**
    * **Purpose:** Runs the Clickstream data generator job.
    * **Roles:** `roles/storage.objectCreator` (to write JSONL files to the GCS Bronze bucket).

* **A custom SA for the Startup Generator:**
    * **Purpose:** Runs the Startup data generator job.
    * **Roles:** `roles/storage.objectCreator` (to write JSONL files to the GCS Bronze bucket).

* **The Master Orchestrator SA:**
    * **Purpose:** The "Master Orchestrator." The most critical SA that runs the main orchestration service.
    * **Roles:**
        * `roles/datastore.user` (to read from Firestore).
        * `roles/storage.objectCreator` (to write Parquet files to GCS Bronze).
        * `roles/dataform.editor` (to create compilations and trigger workflow invocations).
        * `roles/aiplatform.user` (to submit Vertex AI Pipeline jobs).
        * `roles/iam.serviceAccountUser` **(Conditional)**: A binding on the *project* with a condition restricting it *only* to the Google-managed Dataform SA. This is the crucial permission that allows the orchestrator to "act as" the Dataform SA.

* **A user-managed SA for Dataform execution:**
    * **Purpose:** The identity that Dataform *uses* to run its SQL jobs, as specified in the orchestrator's code.
    * **Roles:**
        * `roles/bigquery.dataEditor` (to create/write to tables in the Staging and Warehouse datasets).
        * `roles/bigquery.jobUser` (to run BigQuery jobs).
        * `roles/storage.objectViewer` (to read the raw files from the GCS Bronze bucket).

* **The Google-Managed Dataform SA:**
    * **Purpose:** The default Google-managed SA for Dataform.
    * **Note:** The Master Orchestrator SA is granted "act as" permission on this SA (via IAM condition) to allow it to trigger jobs.

* **A custom SA for Vertex AI Pipelines (Planned):**
    * **Purpose:** The identity for the Vertex AI Pipeline to run as.
    * **Roles:** `roles/bigquery.dataViewer` (to read training data), `roles/bigquery.dataEditor` (to write predictions), `roles/storage.objectAdmin` (to manage pipeline artifacts).

---

## <br> 🚀 4. Core Data Infrastructure

* **Cloud Storage (GCS) Bucket:**
    * **Purpose:** The **Bronze Layer** for the Medallion Architecture.
    * **Folders:** Contains dedicated folders for each of the three data flows to hold raw, immutable data files. Also holds MLOps pipeline artifacts in a separate folder.
* **Cloud Firestore Database:**
    * **Mode:** Native
    * **Collection:** A dedicated collection for the Sales data flow.
    * **Purpose:** The NoSQL source database for sales transactions.
* **BigQuery Datasets:**
    * **A Staging Dataset:** Holds all intermediate tables created by Dataform. This is the **Silver Layer**.
    * **A Warehouse Dataset:** Holds the final, business-ready tables. This is the **Gold Layer**.
        * Contains all final Fact Tables (e.g., Sales, Clickstream Features).
        * Contains all final Dimension Tables (e.g., Customer, Startup).
        * Contains the ML Output Table (e.g., Churn Predictions).

---

## <br> 🚀 5. Batch Ingestion & Orchestration

* **Data Generators (Cloud Run Jobs):**
    1.  **The Sales Generator Job:** Scheduled to run daily. Generates sales data and writes to Firestore.
    2.  **The Clickstream Generator Job:** Scheduled to run daily. Generates clickstream JSONL and writes to GCS.
    3.  **The Startup Generator Job:** Scheduled to run weekly. Generates startup JSONL and writes to GCS.
* **Master Orchestrator (Cloud Run Service):**
    * **Source:** Built from the orchestrator's source code files (Python script, Dockerfile, and requirements file).
    * **Trigger:** A Cloud Scheduler job runs daily to call this service's HTTP endpoint.
    * **Logic:** Executes the full 5-step batch pipeline.

---

## <br> 🚀 6. ELT (Dataform)

* **Repository:** A Dataform repository connected to the project.
* **Workspace:** A dedicated development workspace (e.g., "batch-workspace") that the orchestrator compiles.
* **Logic:** A collection of SQLX files that:
    1.  Create BigQuery External Tables pointing to the GCS Bronze files.
    2.  Clean, type, and pseudonymize data into Staging tables (Silver Layer).
    3.  Aggregate and model data into final Fact and Dimension tables (Gold Layer).

---

## <br> 🚀 7. MLOps Layer  

* **ML Pipeline Spec:** A compiled KFP pipeline file (`.json`) stored in GCS that defines the MLOps pipeline steps.
* **Trigger:** The Master Orchestrator calls the Vertex AI API to submit the training pipeline after Dataform polling succeeds.
* **Output:** The pipeline (when built) will train a churn model and save its predictions to a dedicated ML Output Table in BigQuery.

---

## <br> 🚀 8. Business Intelligence Layer

* **Tool:** Looker Studio
* **Strategy:** A two-phase dashboard:
    1.  **Pre-MLOps:** Connects to the final Fact and Dimension tables to show historical BI ("What happened?").
    2.  **Post-MLOps:** Adds the ML Output Table as a data source to show predictive insights ("What will happen?").
 
---

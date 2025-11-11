# 🚀 Tech Stack

This section details the core technologies, **Google Cloud services**, and supporting infrastructure used to build and operate this 100% Serverless MLOps platform, from data ingestion and transformation to model training.

---

## 🛠️ Core Tech Stack & Languages
A modern, serverless stack leveraging **Python** for data generation and orchestration, and **SQL** for all data transformation pipelines.

---

## ☁️ Google Cloud Services (GCP)
Google Cloud forms the backbone of the platform, providing scalable and "scale-to-zero" services for every step of the MLOps lifecycle.

### Compute & Orchestration
* **Cloud Run (Jobs):** Used for the three serverless, containerized **Data Generator** jobs (Sales, Clickstream, Startup). We only pay for the execution time.
* **Cloud Run (Service):** Hosts the central **"Master Orchestrator"** (`server.py`). It scales to zero when idle and scales up on demand to run the entire batch pipeline.
* **Cloud Scheduler:** Provides the serverless cron triggers that initiate all generator jobs and the Master Orchestrator service.

### Storage & Database
* **Cloud Firestore:** A serverless NoSQL database, used as the "application source" for the Sales data flow.
* **Cloud Storage (GCS):** The central data lake, implementing the **Bronze Layer** for all three data flows. Also used for staging MLOps artifacts (`.json` pipeline specs).

### Data Warehousing & Transformation
* **BigQuery:** The scalable, serverless **Data Warehouse**. Used for external tables, the Staging (Silver) layer, and the final Warehouse (Gold) tables. It's the single source of truth for BI and ML.
* **Dataform:** The serverless, **SQLX-based ELT** engine. It orchestrates the entire transformation pipeline *inside* BigQuery, moving data from Bronze to the final Warehouse tables.

### Machine Learning (MLOps)
* **Vertex AI Pipelines:** The managed service for orchestrating ML workflows. Our Master Orchestrator triggers a pipeline defined by a `.json` spec file.
* **Vertex AI Model Registry (Planned):** The central repository for versioning and managing trained ML models.
* **Vertex AI Batch Prediction (Planned):** The serverless job for running model inference on our entire BigQuery warehouse.

### Business Intelligence
* **Looker Studio:** The free, serverless BI tool used to build and share the **Pre-MLOps (Historical)** and **Post-MLOps (Predictive)** dashboards.

---

## 🐍 Key Python Libraries
These libraries are the building blocks of the Master Orchestrator (`server.py`) and Data Generator (`main.py`) containers.

### Core & Web Framework
* **Flask:** A lightweight web framework used to create the HTTP endpoint for the Master Orchestrator service.
* **Gunicorn:** The production-grade web server used to run the Flask application inside the Cloud Run container.

### Cloud Integration & SDKs
* **google-cloud-firestore:** Used by the Sales Generator to *write* data and by the Orchestrator to *read* data.
* **google-cloud-storage:** Used by the Clickstream/Startup Generators to *write* files and by the Orchestrator to *land* the Sales Parquet file.
* **google-cloud-aiplatform:** The primary **Vertex AI SDK** used by the Orchestrator to programmatically submit the MLOps training pipeline.
* **requests:** Used by the Orchestrator to make HTTP POST/GET requests to the **Dataform API** (to compile, trigger, and poll).
* **google-auth:** Used to handle authentication for all Google Cloud service-to-service API calls.

### Data Handling
* **pandas:** Used by the Orchestrator to pull data from Firestore and convert it into a DataFrame.
* **pyarrow:** The essential library that enables `pandas` to write data in the efficient **Parquet** file format.

---

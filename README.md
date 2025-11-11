# 🚀 100% Serverless GCP Batch Data & MLOps Platform

This repository outlines the blueprint for a **100% serverless, end-to-end Data, ML, & AI Solution Architecture** built on Google Cloud Platform. It showcases expertise across Data Ingestion, tiered ELT, Medallion Architecture, Data Warehousing, and automated MLOps.

This Solution Architecture is focused on ingesting **three distinct batch data flows**, listed below, orchestrating their transformation, and feeding them into an automated machine learning pipeline to make predictions.

 **a) Data about new Start-ups:** API Calls made to external databases every Friday at 1 AM GMT, that contained Start-up related Data that brought in - firstly, a One-time Full Load, then subsequent Incremental Delta Loads that came in once every Friday at 1 AM GMT. 

 **b) Website Clickstreams events:** 1000 Records of Website Clickstreams Data, accumulated over a period of time between 00:00 to 23:59 on that day, sent to the Platform everyday at 1 AM GMT. 

 **c) Sales Transaction Data:** A new batch of Sales Transactions Data, accumulated over the previous day, sent to the Platform every day at 1 AM GMT.

**NOTE:** This repository is only a descriptive write-up of the actual Design and Implementation process followed by the Architect. This repository is not intended to provide readers with the ability to fork, clone or download anything.

---

## 💡 Goals

1.  **Build a Comprehensive Data & MLOps Platform:** Design and implement a complete solution on GCP that handles multiple batch data sources.
2.  **Embrace a 100% Serverless Model:** Utilize "scale-to-zero" services (Cloud Run, Dataform, BigQuery, Firestore) to create a powerful, cost-effective platform with minimal idle costs.
3.  **Demonstrate End-to-End Automation:** Create a "lights-off" pipeline where data ingestion, transformation (ELT), and MLOps training are all triggered automatically by a central orchestrator.

---

## 🚀 Architectural Scope

The Platform processes three distinct, simulated batch data flows, transforming them from raw files and NoSQL entries into a unified, analytics-ready Data Warehouse. This warehouse serves as the foundation for both historical BI (Pre-MLOps) and predictive analytics (Post-MLOps).

| Skill Area | Key GCP Services Demonstrated |
| :--- | :--- |
| **Batch Ingestion** | Cloud Scheduler, Cloud Run Jobs, Cloud Firestore, GCS |
| **Orchestration** | Cloud Run (Service) |
| **Storage, ELT & Modeling** | GCS (Bronze Layer), BigQuery, Dataform |
| **MLOps & Prediction** | Vertex AI Pipelines, Vertex AI Model Registry |
| **Business Intelligence** | Looker Studio (Pre- & Post-MLOps Dashboards) |
| **Security & IAM** | Custom Service Accounts, IAM Conditions |

---

## 🗺️ Component Deep Dive

| Layer | GCP Services | Purpose |
| :--- | :--- | :--- |
| **Triggers** | Cloud Scheduler | Provides scheduled (cron) triggers for all data generation jobs and the master orchestration service. |
| **Data Generation** | Cloud Run Jobs (x3) | Three separate, containerized Python jobs that simulate and generate batch data for Sales, Clickstream, and Startups. |
| **NoSQL Source** | Cloud Firestore | Acts as the "application database" (source of truth) for the Sales data flow. |
| **Orchestration** | Cloud Run (Service) | The **"Master Orchestrator"**. This central service runs the end-to-end pipeline: pulls from Firestore, lands to GCS, triggers Dataform, polls for completion, and triggers the Vertex AI ML pipeline. |
| **Data Lake (Bronze)** | Cloud Storage (GCS) | The Bronze Layer. Stores raw, immutable data files for all three flows (Parquet for Sales, JSONL for Clickstream/Startup) before they are processed by ELT. |
| **ELT (In-Warehouse)**| Dataform | The serverless, SQL-first transformation engine. Manages the entire Medallion pipeline (Bronze-to-Warehouse) directly within BigQuery using version-controlled SQLX files. |
| **Data Warehouse** | BigQuery | The scalable, serverless warehouse. Hosts all Staging and final Warehouse tables (`fct_sales`, `dim_customer`, etc.) and serves as the single source of truth for both BI and MLOps. |
| **MLOps Pipeline** | Vertex AI Pipelines | Provides a reproducible automation framework for model training, versioning, and deployment, feeding predictive insights (e.g., churn scores) back to the DWH. |
| **Business Intelligence**| Looker Studio | The free visualization tool used to build "Pre-MLOps" (historical) and "Post-MLOps" (predictive) dashboards on top of the BigQuery warehouse. |

---

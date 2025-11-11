## 🔮 Future Enhancements (Post-PoC Reliability and Scale)

This section outlines planned improvements to strengthen the reliability, cost-efficiency, and intelligence of the `winter-days` platform.

---

### 🛡️ 1. Cloud Logging, Monitoring & Error Reporting

#### 1.1. Centralized Logging
* Enable **Cloud Logging sinks** to route logs from all Cloud Run Jobs, the Cloud Run Service, Dataform, and Vertex AI jobs into a centralized BigQuery dataset or GCS log bucket.
* Apply **log-based metrics** to track batch run times, data volume ingested, and pipeline success rates.

#### 1.2. Monitoring Dashboards
* Build **Cloud Monitoring dashboards** to visualize:
    * Orchestrator execution time and error rates (e.g., 4xx/5xx).
    * Dataform workflow duration and failure rates.
    * Vertex AI training and batch prediction job runtimes.
    * BigQuery slot utilization during Dataform runs.

#### 1.3. Error Reporting & Alerts
* Enable **Error Reporting** for the Orchestrator service to automatically group stack traces (like the ones we debugged).
* Configure **alerting policies** to send notifications (email or Slack) when:
    * The Master Orchestrator service fails (returns a 500).
    * A Dataform workflow invocation (polled by the orchestrator) returns `FAILED`.
    * The Vertex AI MLOps pipeline job fails.

---

### 💰 2. Plan for GCS Cleanup and Cost Control

#### 2.1. GCS Lifecycle Management
* Apply a **Lifecycle Management Policy** to the GCS Bronze bucket.
* **Rule:** Automatically transition raw files in Folders to a cheaper storage class (e.g., Coldline) after 90 days.
* **Rule:** Automatically delete files older than 365 days (or based on compliance needs).

#### 2.2. System Bucket Cleanup
* Apply a **Lifecycle Policy** to the system-generated bucket to delete old source archives (`.tgz` files) after 60 days to manage cost.

---

### 🤖 3. Intelligent Enhancements (The Vision)

* **Connect to Real-World Sources:** Systematically replace the three data generator jobs with real-world connectors:
    * **Sales:** Connect to the **Stripe** or **Shopify** API, or read from a **Salesforce** data export.
    * **Clickstream:** Replace the generator with the native **Google Analytics (GA4) to BigQuery Export**.
    * **Startup:** Connect to a live API like **Crunchbase** or **PitchBook**.
* **Build the Other ML Models:** Use the rich data in the warehouse to build the other models we planned:
    * **Startup Growth Model:** Use `dim_startup` to predict a startup's success.
    * **Fraud Detection Model:** Use `fct_sales` data to build a real-time fraud detection pipeline.
* **Deploy AI Agents:** Build the **"BI Bot" (Vertex AI Agent)** we discussed. This agent would be pointed at the Data Warehouse and allow a sales manager to ask in plain English, "Show me my top 5 customers in the 'North' region who are at high risk of churn."

---

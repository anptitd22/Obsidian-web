---
title: Welcome to My Channel
---

# Tran Duc An

[GitHub](https://github.com/anptitd22) | [LinkedIn](https://www.linkedin.com/in/tran-duc-an/) | [Facebook](https://www.facebook.com/an.badaoc/) | [tu123098b@gmail.com](mailto:tu123098b@gmail.com) | [+84.97.120.9142](tel:+84971209142)

---

## Summary
Data Engineer with 1 year of experience specializing in automated ETL pipelines and modern Lakehouse architectures. Proven expertise in optimizing high-performance crawling systems and executing complex cloud-to-on-premise migrations to minimize operational costs. Committed to mastering scalable infrastructure and advanced data modeling to deliver high-integrity solutions, with a long-term focus on becoming a Data Architect.

---

## Work Experience

### **Data Engineer**, TOPCV Vietnam Joint Stock Company
*Sep 2025 – Present*

* Operate, maintain, and optimize the Lakehouse ecosystem (Airflow, Kafka, Trino, Hive, MinIO, Iceberg, OPA, OpenMetadata, RDBMS) while driving infrastructure migrations from Google Cloud Platform to on-premise environments; leverage Grafana and ArgoCD to monitor cluster health, track deployment sync statuses, and ensure high system reliability.
* **Lakehouse/Cloud Operations & Ecosystem Maintenance:**
  * Build and operate OpenMetadata to centralize data discovery, automated end-to-end data lineage across the Lakehouse, and streamlined metadata ingestion pipelines for enterprise-wide data governance.
  * Build and operate Open Policy Agent (OPA) and Rego policy logic; enforce fine-grained access control across all data hierarchy levels to govern PII data access and grant tailored dataset permissions for cross-functional teams.
  * Build and migrate Apache Airflow from Bitnami (v2.3.3) to the official distribution (v3.1.6), optimizing DAG performance to eliminate parsing errors and reduce scheduler overhead.
  * Engineer automated ETL pipelines to sync GA4 event streams and Google Search Console performance data from BigQuery into an on-premise lakehouse via Trino, enabling cross-system data verification and comparative analytics against the in-house tracking log system.
  * Optimize PowerBI dashboard queries via Trino by 4x by staging pre-aggregated data into a dedicated serving layer.
  * Design and implement data visualizations via Apache Superset, enabling self-service analytics and automated reporting for teams, focusing on pipeline metrics and service costs.
* **Automated Data Collection & Crawling Systems:**
  * Develop high-performance ingestion pipelines using Asynchronous logic and multi-proxy rotation; leverage Airflow's dynamic pod creation to parallelize tasks across a K8s cluster, boosting execution speed by 70% with automated recovery.
* **Data Warehousing & Business Automation:**
  * Engineer scalable pipeline systems to collect, ETL/ELT and analyze tax codes (TIN) and SEO metrics, automating cross-departmental reporting and reducing manual operational workflows.
  * Contribute to building a corporate Data Mart utilizing Star Schema and SCD Type 1 logic, streamlining business reporting and ensuring high data integrity for downstream analytics.

### **Data Engineer (Collaborator)**, Center for Digital Innovation and Technology in PTIT
*Jun 2026 – Aug 2026*

* Researched and supported the implementation of a lightweight game tracking log system leveraging Python and Redis to ingest real-time logs and sync GA4/Firebase app events into MongoDB.

---

## Projects

### **Data Platform** | [GitHub Repository](https://github.com/anptitd22/DataPlatform)
Architected an enterprise-grade data lakehouse platform with ACID compliance, creating deployment templates for Google Cloud VMs to automate infrastructure setup and enable scalable analytics.

* **Data Pipelines & Analytics Products (Built on Platform):**
  * **CDC Ingestion:** Engineered low-latency streaming pipelines utilizing Kafka (KRaft mode) to capture change data capture (CDC) events from MongoDB directly into Apache Iceberg table format.
  * **Tracking Log Streaming:** Developed a custom log generation simulator publishing streaming events to Kafka; implemented PySpark Structured Streaming jobs on the Spark cluster to ingest, process, and append micro-batch logs into Iceberg tables.
  * **Data Mart:** Orchestrated batch ELT pipelines processing raw CSV datasets into Silver & Gold-layer data marts (Star Schema with SCD Type 1 logic) using PyIceberg, PyArrow, and Trino; integrated Apache Superset for interactive dashboards and decision-making analytics.

---

## Education

**Posts and Telecommunications Institute of Technology** — Computer Science  
*2022 – 2027*  
* **GPA:** 3.2/4.0

---

## Certificates

* **TOEIC:** 630 (2026)

---

## Skills

* **Lakehouse:** Iceberg, MinIO, Trino, Hive, Spark, Kafka
* **Orchestration:** Airflow, CronJob
* **Data Visualization & Monitoring:** Superset, Grafana
* **Data Governance & Authorization:** OpenMetadata, Open Policy Agent
* **Databases:** MySQL, PostgreSQL, MongoDB
* **Data Crawling:** Selenium, Playwright
* **Cloud:** Google Cloud Platform
* **Data Modeling:** Star Schema
* **Design System:** Async
* **Programming Languages:** Python
* **DevOps & Infrastructure:** Docker, Kubernetes, ArgoCD
* **Others:** Git, Jira, Linux

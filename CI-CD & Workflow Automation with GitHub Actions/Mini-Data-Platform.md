# Mini Data Platform (Dockerized)
## Project Goal
Build a functional, end-to-end data platform using **Docker Compose** to manage the full data lifecycle: Collection, Processing, Storage, and Visualization.

---

## 🏗 Architecture & Components
The platform consists of four primary services running in isolated Docker containers:

| Component | Technology | Purpose |
| :--- | :--- | :--- |
| **Database** | PostgreSQL | Relational storage for processed, structured data. |
| **Processing** | Apache Airflow | Workflow orchestration and data pipeline execution. |
| **Storage** | MinIO | S3-compatible object storage for raw CSV/file ingestion. |
| **Dashboards** | Metabase | Business intelligence tool for charts and reporting. |

---

## 🚀 Project Workflow

### Part 1: Infrastructure Setup
* Define all four services in a single `docker-compose.yml` file.
* Ensure persistent storage using Docker volumes.
* Configure network bridges to allow Airflow to communicate with MinIO and PostgreSQL.

### Part 2: Data Engineering Pipeline
* **Ingestion:** Create a sample data generator to produce synthetic sales or user data.
* **Orchestration:** Develop an Airflow DAG that:
    1. Detects new files in **MinIO**.
    2. Performs data cleaning and transformation.
    3. Loads the final dataset into **PostgreSQL**.

### Part 3: Data Visualization
* Connect **Metabase** to the PostgreSQL instance.
* Design a dashboard showing key performance indicators (KPIs) and trends.

---

## 🛠 CI/CD & Automation (GitHub Actions)
The repository must include a `.github/workflows/main.yml` file to automate the following:

* **Continuous Integration (CI):** * Build and lint Docker images for each service on every commit.
* **Continuous Deployment (CD):** * Automatically deploy updated containers to a designated test environment.
* **Data Flow Validation:** * Execute automated integration tests ensuring data successfully moves: 
      `MinIO (Ingestion)` → `Airflow (Processing)` → `PostgreSQL (Storage)` → `Metabase (API check)`.

---

## 📁 Repository Structure
```text
├── dags/                # Airflow DAG definitions
├── data_generator/      # Python scripts for synthetic data
├── docker-compose.yml   # Platform orchestration
├── config/              # Configuration files (Postgres/Airflow)
├── .github/workflows/   # CI/CD pipelines
├── .gitignore           # Excluding volumes, logs, and .env
└── README.md            # Setup instructions and documentation

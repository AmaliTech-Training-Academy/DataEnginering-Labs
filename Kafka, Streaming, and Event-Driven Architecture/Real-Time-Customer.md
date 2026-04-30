# Real-Time Customer Heart Beat Monitoring System

## Project Overview
This project aims to simulate and process real-time heart rate data for customers using a data pipeline. You'll learn how to generate synthetic data, stream it using **Apache Kafka**, and store it in a **PostgreSQL** database. This hands-on project introduces you to important concepts like data simulation, message queuing, real-time processing, and database integration—core components of modern data engineering systems.

## Project Objectives
By the end of this project, students will be able to:
1. **Simulate** real-time sensor data to mimic heart rate monitors.
2. **Use Kafka** for real-time data streaming, setting up producers and consumers.
3. **Process** incoming data with basic validation and store it in a structured relational database.
4. **Design and implement** a PostgreSQL database schema suitable for time-series data.
5. **Build and test** a simple data pipeline, understanding each component’s role.
6. **Optionally visualize** data using a basic dashboard.

---

## Core Components

### 1. Synthetic Data Generator
* Write a Python script that randomly generates heart beat data for multiple fake customers.
* **Data fields:** `customer_id`, `timestamp`, and `heart_rate`.

### 2. Kafka Producer
* Sends the generated data to a Kafka topic.
* Configure it to send messages continuously, simulating a real-time feed.

### 3. Kafka Consumer
* Listens to the Kafka topic, reads the messages, and optionally filters or checks for anomalies (e.g., heart rate too high/low).
* Writes valid data into a PostgreSQL table.

### 4. PostgreSQL Database
* Create a simple schema to store heart beat records with indexing on timestamps for efficient querying.
* Use SQL for inserting and querying historical data.

---

## Project Steps

### 1. Data Simulation & Ingestion
* Develop and run a Python script that continuously creates random but realistic heart beat readings.
* Connect this script to a Kafka producer.

### 2. Real-Time Data Processing
* Set up Kafka and a consumer app (in Python) that reads messages from the topic.
* Process or clean data before sending it to the database.

### 3. Data Storage
* Design a table in PostgreSQL to hold the customer heart beat data.
* Write the logic to insert the processed data into the table.

### 4. System Setup and Testing
* Use `docker-compose` (optional) to run Kafka, Zookeeper, and PostgreSQL locally.
* Write test scripts to simulate different types of heart rate data and check if the full pipeline works.

### 5. Documentation & Reporting
* Document your system architecture with diagrams (e.g., [draw.io](http://draw.io)).
* Write clear setup instructions for running the system.
* Include screenshots of terminal outputs and database tables showing successful ingestion.

### 6. Optional Extensions
* Build a basic dashboard with tools like:
    * **Grafana** (with PostgreSQL as a data source)
    * **Dash** or **Streamlit** (Python-based)

---

## Project Deliverables
* **Python scripts:** Data generator, Kafka producer, and consumer.
* **SQL file:** To create the database schema.
* **Docker Compose file:** (Optional but recommended).
* **ReadMe file:** With setup guide and documentation.
* **Data flow diagram:** (PDF or image).
* **Sample data outputs:** Or screenshots of the running system.
* **(Optional) Dashboard:** Interface or screenshots.

# Real-Time Data Ingestion Using Spark Structured Streaming & PostgreSQL

## Project Overview

In this project, you'll build a real-time data pipeline that simulates an e-commerce platform tracking user activity. You'll generate fake user events (like product views and purchases), stream them using Apache Spark Structured Streaming, and store the processed data in a PostgreSQL database.

You'll gain practical experience with:

- Real-time data streaming
- Data transformation using Spark
- Writing to a relational database
- Handling continuously arriving data

---

## Project Objectives

By the end of this project, you will be able to:

- Simulate and ingest streaming data
- Use Spark Structured Streaming to process data in real time
- Store and verify processed data in a PostgreSQL database
- Understand the architecture of a real-time data pipeline
- Measure and evaluate system performance

---

## Tools & Technologies

- Apache Spark Structured Streaming
- PostgreSQL
- Python (for data generation)
- SQL (for database setup)

---

## Project Structure

### Part 1: Simulate Data

- Write a script that creates fake e-commerce events (CSV files).
- These events include user actions like `"view"` or `"purchase"`, along with product info and timestamps.

### Part 2: Stream with Spark

- Use Spark Structured Streaming to monitor the folder where CSVs are saved.
- Read and process new CSV files as they appear.
- Apply any needed transformations (e.g., data cleaning, type conversion).

### Part 3: Store in PostgreSQL

- Set up a PostgreSQL database and create a table to store incoming events.
- Connect Spark to the database.
- Insert processed data into the table in real time.

---

## Deliverables

| Deliverable | Description |
|---|---|
| `data_generator.py` | Python script to generate CSV event data |
| `spark_streaming_to_postgres.py` | Spark Structured Streaming job to process and write data |
| `postgres_setup.sql` | SQL script to create database and table |
| `postgres_connection_details.txt` | Text file with host, port, user, and password |
| `project_overview.md` | Short write-up explaining your system's components and flow |
| `user_guide.md` | Step-by-step instructions on how to run your project |
| `test_cases.md` | Manual test plan with expected vs actual outcomes |
| `performance_metrics.md` | Report with system performance data (latency, throughput, etc.) |
| `system_architecture.png` | Diagram showing data flow and components |

---

## What to Test

- Are the CSV files being generated correctly?
- Is Spark detecting and processing new files?
- Are the data transformations correct?
- Is data being written into PostgreSQL without errors?
- Are performance metrics (like processing speed) within expected limits?

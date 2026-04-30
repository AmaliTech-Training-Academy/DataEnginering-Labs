# Healthcare Analytics Lab: OLTP to Star Schema

**Data Engineering Mini Project**

---

## Project Overview

You've joined HealthTech Analytics as a junior data engineer. The clinical team built a normalized transactional database (3NF), but analytics queries are slow. Your job: analyze the OLTP schema, identify performance issues, then design and build an optimized star schema.

This mirrors real-world data engineering work.

---

## Part 1: Normalized OLTP Schema

The production system uses 8 normalized tables. Study the schema and understand how data is organized.

```sql
CREATE TABLE patients (
    patient_id INT PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    date_of_birth DATE,
    gender CHAR(1),
    mrn VARCHAR(20) UNIQUE
);

CREATE TABLE specialties (
    specialty_id INT PRIMARY KEY,
    specialty_name VARCHAR(100),
    specialty_code VARCHAR(10)
);

CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100),
    floor INT,
    capacity INT
);

CREATE TABLE providers (
    provider_id INT PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    credential VARCHAR(20),
    specialty_id INT,
    department_id INT,
    FOREIGN KEY (specialty_id) REFERENCES specialties(specialty_id),
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

CREATE TABLE encounters (
    encounter_id INT PRIMARY KEY,
    patient_id INT,
    provider_id INT,
    encounter_type VARCHAR(50), -- 'Outpatient', 'Inpatient', 'ER'
    encounter_date DATETIME,
    discharge_date DATETIME,
    department_id INT,
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (provider_id) REFERENCES providers(provider_id),
    FOREIGN KEY (department_id) REFERENCES departments(department_id),
    INDEX idx_encounter_date (encounter_date)
);

CREATE TABLE diagnoses (
    diagnosis_id INT PRIMARY KEY,
    icd10_code VARCHAR(10),
    icd10_description VARCHAR(200)
);

CREATE TABLE encounter_diagnoses (
    encounter_diagnosis_id INT PRIMARY KEY,
    encounter_id INT,
    diagnosis_id INT,
    diagnosis_sequence INT,
    FOREIGN KEY (encounter_id) REFERENCES encounters(encounter_id),
    FOREIGN KEY (diagnosis_id) REFERENCES diagnoses(diagnosis_id)
);

CREATE TABLE procedures (
    procedure_id INT PRIMARY KEY,
    cpt_code VARCHAR(10),
    cpt_description VARCHAR(200)
);

CREATE TABLE encounter_procedures (
    encounter_procedure_id INT PRIMARY KEY,
    encounter_id INT,
    procedure_id INT,
    procedure_date DATE,
    FOREIGN KEY (encounter_id) REFERENCES encounters(encounter_id),
    FOREIGN KEY (procedure_id) REFERENCES procedures(procedure_id)
);

CREATE TABLE billing (
    billing_id INT PRIMARY KEY,
    encounter_id INT,
    claim_amount DECIMAL(12, 2),
    allowed_amount DECIMAL(12, 2),
    claim_date DATE,
    claim_status VARCHAR(50),
    FOREIGN KEY (encounter_id) REFERENCES encounters(encounter_id),
    INDEX idx_claim_date (claim_date)
);
```

### Sample Data

```sql
INSERT INTO specialties VALUES
    (1, 'Cardiology', 'CARD'),
    (2, 'Internal Medicine', 'IM'),
    (3, 'Emergency', 'ER');

INSERT INTO departments VALUES
    (1, 'Cardiology Unit', 3, 20),
    (2, 'Internal Medicine', 2, 30),
    (3, 'Emergency', 1, 45);

INSERT INTO providers VALUES
    (101, 'James', 'Chen', 'MD', 1, 1),
    (102, 'Sarah', 'Williams', 'MD', 2, 2),
    (103, 'Michael', 'Rodriguez', 'MD', 3, 3);

INSERT INTO patients VALUES
    (1001, 'John', 'Doe', '1955-03-15', 'M', 'MRN001'),
    (1002, 'Jane', 'Smith', '1962-07-22', 'F', 'MRN002'),
    (1003, 'Robert', 'Johnson', '1948-11-08', 'M', 'MRN003');

INSERT INTO diagnoses VALUES
    (3001, 'I10', 'Hypertension'),
    (3002, 'E11.9', 'Type 2 Diabetes'),
    (3003, 'I50.9', 'Heart Failure');

INSERT INTO procedures VALUES
    (4001, '99213', 'Office Visit'),
    (4002, '93000', 'EKG'),
    (4003, '71020', 'Chest X-ray');

INSERT INTO billing VALUES
    (14001, 7001, 350, 280, '2024-05-11', 'Paid'),
    (14002, 7002, 12500, 10000, '2024-06-08', 'Paid');

INSERT INTO encounters VALUES
    (7001, 1001, 101, 'Outpatient', '2024-05-10 10:00:00', '2024-05-10 11:30:00', 1),
    (7002, 1001, 101, 'Inpatient', '2024-06-02 14:00:00', '2024-06-06 09:00:00', 1),
    (7003, 1002, 102, 'Outpatient', '2024-05-15 09:00:00', '2024-05-15 10:15:00', 2),
    (7004, 1003, 103, 'ER', '2024-06-12 23:45:00', '2024-06-13 06:30:00', 3);

INSERT INTO encounter_diagnoses VALUES
    (8001, 7001, 3001, 1), (8002, 7001, 3002, 2),
    (8003, 7002, 3001, 1), (8004, 7002, 3003, 2),
    (8005, 7003, 3002, 1), (8006, 7004, 3001, 1);

INSERT INTO encounter_procedures VALUES
    (9001, 7001, 4001, '2024-05-10'), (9002, 7001, 4002, '2024-05-10'),
    (9003, 7002, 4001, '2024-06-02'), (9004, 7003, 4001, '2024-05-15');
```

---

## Part 2: Find the Performance Problem

You're given 4 business questions. Write the SQL to answer each one using the normalized schema above. Run the queries, measure performance, and identify bottlenecks.

### Question 1: Monthly Encounters by Specialty

**What we need:** For each month and specialty, show total encounters and unique patients by encounter type.

**Your task:**
1. Write the SQL query
2. Document: How many tables do you join?
3. Measure: What's the query execution time?
4. Analyze: Why is this slow? *(Hint: JOIN chain + GROUP BY)*

### Question 2: Top Diagnosis-Procedure Pairs

**What we need:** What are the most common diagnosis-procedure combinations? Show the ICD code, procedure code, and encounter count.

**Your task:**
1. Write the SQL query
2. Document: How do you join diagnosis and procedure data across separate junction tables?
3. Measure: What's the query execution time?
4. Analyze: Why is this slow? *(Hint: Two junction tables → row explosion)*

### Question 3: 30-Day Readmission Rate

**What we need:** Which specialty has the highest readmission rate? (Definition: inpatient discharge, then return within 30 days)

**Your task:**
1. Write the SQL query (you'll need a self-join on encounters)
2. Document: How do you detect a readmission using `discharge_date` and subsequent encounters?
3. Measure: What's the query execution time?
4. Analyze: Why is this slow? *(Hint: Self-joins on large tables)*

### Question 4: Revenue by Specialty & Month

**What we need:** Total allowed amounts by specialty and month. Which specialties generate most revenue?

**Your task:**
1. Write the SQL query
2. Document: What's the JOIN chain? (`billing → encounters → providers → specialties`)
3. Measure: What's the query execution time?
4. Analyze: Why is this slow? *(Hint: Multiple JOINs + aggregation)*

### Part 2 Deliverable: `query_analysis.txt`

For each question, document using this format:

    QUESTION #: [Title]

    SQL Query:
    [Your query here]

    Schema Analysis:
    - Tables joined: [list them]
    - Number of joins: [X]

    Performance:
    - Execution time: [X seconds]
    - Estimated rows scanned: [Y]

    Bottleneck Identified:
    [Why is this slow? What's the root cause?]

---

## Part 3: Design the Star Schema

Now that you've experienced the performance pain, design an optimized dimensional model.

### 3.1: Design Decisions

Answer these design questions in a document called `design_decisions.txt`:

**Decision 1: Fact Table Grain**

Choose one:
- Option A: One row per encounter
- Option B: One row per diagnosis within an encounter
- Option C: One row per procedure within an encounter

Document your choice and why it's best for these 4 queries.

**Decision 2: Dimension Tables**

List which dimension tables you'll create and what attributes each should contain:
- Date dimension (what columns?)
- Patient dimension (what columns?)
- Provider dimension (what columns?)
- Specialty dimension (what columns?)
- Department dimension (what columns?)
- Encounter type dimension (what columns?)
- Others? (diagnoses? procedures?)

**Decision 3: Pre-Aggregated Metrics**

What metrics should be stored directly in the fact table to avoid expensive joins?
- `diagnosis_count` — how many diagnoses per encounter?
- `procedure_count` — how many procedures per encounter?
- `total_allowed` — sum of billing amounts?
- Others?

Justify why pre-aggregating these metrics helps performance.

**Decision 4: Bridge Tables**

Will you use bridge tables for many-to-many relationships (diagnoses-to-encounters, procedures-to-encounters)? Why or why not?

### 3.2: Build the Star Schema

Create complete DDL for your star schema in `star_schema.sql`:

**Dimension tables:**
- `dim_date` — date_key, calendar_date, year, month, quarter, etc.
- `dim_patient` — patient_key, patient_id, first_name, age_group, etc.
- `dim_provider` — provider_key, provider_id, name, specialty, etc.
- `dim_specialty` — specialty_key, specialty_id, specialty_name
- `dim_department` — department_key, department_id, department_name
- `dim_encounter_type` — encounter_type_key, type_name

**Fact table:**
- `fact_encounters` — all necessary foreign keys, pre-aggregated metrics, indexes

**Bridge tables (if needed):**
- `bridge_encounter_diagnoses`
- `bridge_encounter_procedures`

**Include:**
- Primary keys (surrogate keys for dimensions)
- Foreign key relationships
- Appropriate indexes
- Comments explaining each table's purpose

### 3.3: Translate Queries to Star Schema

Rewrite each of the 4 queries to use your new star schema. Document in `star_schema_queries.txt`.

For each query, show:
1. The SQL (optimized for star schema)
2. Execution time estimate (e.g., `~150ms vs. 1.8s original`)
3. Improvement factor (e.g., `12x faster`)
4. Explanation: Why is it faster? (fewer joins? pre-aggregated? denormalization? better indexes?)

### 3.4: ETL Logic

Document your ETL approach in `etl_design.txt`. Write pseudocode or narrative describing:

**1. Dimension Load Logic**
- How do you populate `dim_patient` from the `patients` table?
- How do you populate `dim_date` (one-time load)?
- How do you handle updates to dimensions?

**2. Fact Table Load Logic**
- For each encounter, how do you look up dimension keys?
- How do you calculate pre-aggregated metrics?
- How do you handle missing data?

**3. Bridge Table Load Logic**
- How do you populate `bridge_encounter_diagnoses`?
- How do you populate `bridge_encounter_procedures`?

**4. Refresh Strategy**
- How often would you load? (daily? incremental? full refresh?)
- How would you handle late-arriving facts?

---

## Part 4: Analysis & Reflection

Write 1–2 pages in `reflection.md` addressing:

**1. Why Is the Star Schema Faster?**
- Compare number of JOINs in normalized vs. star schema
- Where is data pre-computed in the star schema?
- Why does denormalization help analytical queries?

**2. Trade-offs: What Did You Gain? What Did You Lose?**
- What did you give up? (data duplication, ETL complexity)
- What did you gain? (faster queries, simpler analysis)
- Was it worth it?

**3. Bridge Tables: Worth It?**
- Why keep diagnoses/procedures in bridge tables instead of denormalizing into fact?
- What's the trade-off?
- Would you do it differently in production?

**4. Performance Quantification**

Show numbers for at least 2 queries:
- Original execution time: X seconds
- Optimized execution time: Y seconds
- Improvement: X/Y = ? times faster
- Main reason for the speedup

---

## Deliverables Checklist

| File | Description |
|---|---|
| `query_analysis.txt` | 4 OLTP queries with performance analysis |
| `design_decisions.txt` | All star schema design choices documented |
| `star_schema.sql` | Complete DDL for star schema |
| `star_schema_queries.txt` | 4 optimized queries with performance comparison |
| `etl_design.txt` | ETL pseudocode/narrative |
| `reflection.md` | 1–2 page analysis and reflection |

# PII Detection & Data Quality Validation Pipeline
## Data Engineering Mini Project

### 1. Project Overview
You've joined a fintech company that just collected customer data from multiple sources. It's messy and contains sensitive information. Your job: build a Python pipeline to profile the data, detect PII, validate quality, and remediate issues.

> **Note:** This is real work—data engineers spend ~40% of their time cleaning data before analysis.

---

### 2. The Dataset
**File:** `customers_raw.csv`  
**Expected Schema:** `customer_id, first_name, last_name, email, phone, date_of_birth, address, income, account_status, created_date`

#### Column Type & Validation Rules
| Column | Type | Validation Rules |
| :--- | :--- | :--- |
| **customer_id** | Integer | Unique, positive |
| **first_name** | String | Non-empty, 2-50 chars, alphabetic |
| **last_name** | String | Non-empty, 2-50 chars, alphabetic |
| **email** | String | Valid email format |
| **phone** | String | Valid phone (normalize format) |
| **date_of_birth** | Date | Valid date, `YYYY-MM-DD` |
| **address** | String | Non-empty, 10-200 chars |
| **income** | Numeric | Non-negative, ≤ $10M |
| **account_status** | String | Must be: 'active', 'inactive', 'suspended' |
| **created_date** | Date | Valid date, `YYYY-MM-DD` |

---

### 3. Project Parts

#### Part 1: Exploratory Data Quality Analysis
**Objective:** Profile raw data to understand what's broken.
* **Completeness:** Which columns have missing values? (Report as %)
* **Data types:** Verify if types are correct (e.g., is `date_of_birth` a string or datetime?).
* **Format issues:** Identify non-standard phone and date formats.
* **Uniqueness:** Check if `customer_id` is unique.
* **Invalid values:** Find "invalid_date", negative incomes, or ages > 150.
* **Categorical validity:** Validate `account_status` values.

**Deliverable:** `data_quality_report.txt`

---

#### Part 2: Detect PII
**Objective:** Identify personally identifiable information.
* **PII Identification:** Names, contact info, and sensitive personal data (DOB, address).
* **Regex Detection:** Use regex for email (`something@domain.com`) and phone patterns.
* **Quantify Risk:** Count occurrences and assess impact of a potential breach.

**Deliverable:** `pii_detection_report.txt`

---

#### Part 3: Build a Data Validator
**Objective:** Define and apply validation rules.
* **Tooling:** Choose a library (Pandera, Pydantic, Great Expectations, or custom).
* **Validation:** Apply rules from the table in Section 2.
* **Reporting:** Document specific row/column failures.

**Deliverable:** `validation_results.txt`

---

#### Part 4: Clean the Data
**Objective:** Fix data quality issues.
* **Normalization:** * Phone → `XXX-XXX-XXXX`
    * Dates → `YYYY-MM-DD`
    * Names → Title Case
* **Missing Values:** Implement a strategy (delete, placeholder, or flag).
* **Post-Validation:** Re-run validators to confirm fixes.

**Deliverable:** `cleaning_log.txt` and `customers_cleaned.csv`

---

#### Part 5: Mask PII
**Objective:** Protect sensitive data before sharing.
* **Implement Masking:**
    * Names: `John Doe` → `J***`
    * Emails: `john.doe@gmail.com` → `j***@gmail.com`
    * Phone: `555-123-4567` → `***-***-4567`
    * Address: `[MASKED ADDRESS]`
    * DOB: `1985-03-15` → `1985-**-**`
* **Output:** Save as `customers_masked.csv`.

**Deliverable:** `masked_sample.txt` (Before/After comparison)

---

#### Part 6: Build an End-to-End Pipeline
**Objective:** Orchestrate all steps into a single automated workflow.
* **Script Logic:** Load → Clean → Validate → Detect PII → Mask → Save → Report.
* **Production Standards:** Include error handling, logging, and documentation.

**Deliverable:** `pipeline_execution_report.txt`

---

#### Part 7: Reflection & Governance
**Objective:** Think critically about data quality and privacy.
Write a 1-2 page reflection addressing:
1. **Top 5 DQ Issues:** Identification, fix, and impact.
2. **Risk Assessment:** Sensitivity of detected PII.
3. **Masking Trade-offs:** Utility vs. Privacy.
4. **Validation Strategy:** Effectiveness of the chosen rules.
5. **Production Operations:** Scheduling and failure handling.
6. **Lessons Learned.**

**Deliverable:** `reflection.md`

---

### 4. Deliverables Checklist
- [ ] `data_quality_report.txt`
- [ ] `pii_detection_report.txt`
- [ ] `validation_results.txt`
- [ ] `cleaning_log.txt`
- [ ] `masked_sample.txt`
- [ ] `pipeline_execution_report.txt`
- [ ] `customers_cleaned.csv`
- [ ] `reflection.md`
- [ ] Python source code files

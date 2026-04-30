# Power BI Dashboard for Tracking Student Progress and Performance: Dare Careers

## Project Overview: Preamble

Dare Careers is a business that provides training in Power BI and AWS Cloud. To effectively monitor the progress and performance of its students, Dare Careers requires a comprehensive Power BI dashboard. This dashboard will track key metrics, including daily Zoom attendance records, daily participation, weekly quiz and lab grades, and learner status (graduation and certification).

### Data Organisation

| Data Type | Structure |
|---|---|
| **Zoom Attendance** | Directories for Power BI and AWS Cloud tracks, each containing 10 sub-directories (Week 1–10) with daily attendance records |
| **Lab and Quizzes** | Directories for both tracks with weekly lab and quiz grades across the 10-week period |
| **Participation Records** | Directories for both tracks with daily participation records per learner over 10 weeks |
| **Status Records** | Directories for both tracks with certification and graduation status per learner |

This dashboard will provide insights into learner engagement, progress, and outcomes — helping program managers and trainers track performance trends, identify at-risk learners, and measure key metrics such as certification and graduation rates.

---

## Page 1: Overall Performance Metrics

### Objective

Provide a high-level overview of learner performance, summarising key metrics for program stakeholders.

### Visuals

#### Bar Charts

- **Graduation Rates** — Percentage of learners who graduated from each cohort
- **Certification Rates** — Percentage of learners who achieved certification
- **Dropout Rates** — Percentage of learners who dropped out of the program
- **Average Attendance** — Average percentage of total class time attended by learners
- **Average Participation** — Average engagement score based on daily participation
- **Average Assessment Scores** — Average quiz and lab scores for each cohort

#### Cards (Summary Figures)

- **Total Certifications** — Number of learners who achieved certification
- **Total Learners** — Total number of learners enrolled across all cohorts
- **Total Dropouts** — Number of learners who dropped out before completing the program
- **Total Graduations** — Total number of learners who successfully graduated

### Filters (Slicers)

| Slicer | Options |
|---|---|
| **Cohort** | Filter by specific cohorts |
| **Track** | Power BI / AWS Cloud |
| **Certification Type** | Filter by certification type (if applicable) |
| **Learner Status** | Certified / Not Certified |

---

## Page 2: Detailed Learner Insights

### Objective

Provide detailed insights into individual learner progress and engagement for trainers and program managers.

### Visuals

#### Learner Table

Display a table listing all learners with their respective tracks and key performance metrics:
- Attendance
- Participation
- Assessment scores

#### Cards (Summary Figures)

- **Count of Labs** — Total number of labs completed by all learners
- **Average Labs Completed** — Average number of labs completed per learner
- **Total Hours Spent in Class** — Total class attendance hours per learner
- **Average Attendance Rate** — Average attendance rate per learner
- **Average Participation Rate** — Average daily engagement rate per learner
- **Average Assessment Scores** — Average score across all quizzes and labs per learner

### Filters (Slicers)

| Slicer | Options |
|---|---|
| **Cohort** | Filter by individual cohorts |
| **Track** | Power BI / AWS Cloud |
| **Month** | Filter by month within the 10-week program |
| **Week** | Filter by specific weeks |
| **Learner Status** | Certified / Not Certified |
| **Program Status** | Ongoing / Completed |

---

## Additional Instructions

### 1. Class Attendance Rule

> A learner is marked as **"Attended"** only if they spent **more than 30 minutes** in a Zoom session.

This condition applies to all attendance calculations across both pages.

### 2. Data Preparation

- Ensure daily attendance records, participation records, lab and quiz grades, and learner status data are structured and cleaned before importing into Power BI.
- Use daily attendance directories to track compliance with the 30-minute attendance condition.
- Aggregate weekly lab and quiz scores to provide averages and totals for both learner-level and cohort-level analysis.

### 3. Modelling and DAX

- Use DAX formulas to dynamically calculate:
  - Attendance percentages
  - Averages (participation, scores)
  - Certification and graduation rates
- Ensure relationships between the following tables are properly defined:
  - Attendance
  - Participation
  - Labs
  - Quizzes
  - Learner Status

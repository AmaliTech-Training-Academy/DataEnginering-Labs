# Flight Fare Prediction Using Machine Learning

## Project Overview

This project challenges you to build an end-to-end machine learning pipeline for predicting flight fares based on key travel and airline information.

You will apply the full data science workflow — from problem framing and data cleaning to model training, evaluation, and optimization. You'll work independently to design, structure, and implement each step of the pipeline while documenting your process and results.

---

## Project Objectives

- **Problem Framing:** Define a business question and translate it into a machine learning regression problem.
- **Data Cleaning & Transformation:** Prepare and preprocess real-world flight price data for modeling.
- **Exploratory Data Analysis (EDA):** Explore and visualize trends in airline fares, routes, and seasonal pricing.
- **Model Development:** Build and evaluate multiple regression models to predict total fare.
- **Model Optimization:** Apply hyperparameter tuning, regularization, and cross-validation to improve model accuracy.
- **Model Interpretation:** Explain feature importance and interpret what drives fare variations.
- **Insights & Reporting:** Present findings through visualizations and structured documentation.

---

## Project Steps

### Step 1: Problem Definition & Data Understanding

1. **Understand the business goal**
   Airlines and travel platforms want to estimate ticket prices based on route, airline, and travel date to support pricing strategy and dynamic recommendations.

2. **Define the ML task**
   - Type: Supervised Regression
   - Target variable: `Total Fare`
   - Features: `Airline`, `Source`, `Destination`, `Date`, `Base Fare`, `Tax & Surcharge`, etc.

3. **Load the dataset**
   - Use `Flight_Price_Dataset_of_Bangladesh.csv` *(available on Kaggle)*
   - Inspect the structure using pandas: `.info()`, `.describe()`, `.head()`

4. **Initial observations**
   - Note missing data, outliers, categorical columns, and numerical ranges.
   - Document assumptions and limitations.

---

### Step 2: Data Cleaning & Preprocessing

#### A. Cleaning Data

1. Drop irrelevant or duplicate columns if present (e.g., `"Unnamed"`, `"Index"`).
2. Handle missing values:
   - **Numerical:** Impute using mean/median or drop rows with excessive missingness.
   - **Categorical:** Impute with `"Unknown"` or mode.
3. Correct invalid entries:
   - Negative fares or base fares → replace or remove.
   - Inconsistent city names → normalize (e.g., `"Dhaka"`, `"Dacca"`).
4. Validate data types:
   - Convert numeric columns (`Base Fare`, `Tax & Surcharge`, `Total Fare`) to `float`.
   - Convert date columns to `datetime`.

#### B. Feature Engineering

1. Create new features:
   - `Total Fare = Base Fare + Tax & Surcharge` (if missing)
   - Extract `Month`, `Day`, `Weekday`, and `Season` from `Date`
2. Encode categorical variables:
   - Use One-Hot Encoding (`pd.get_dummies`) or Label Encoding for `Airline`, `Source`, and `Destination`
3. Scale numerical features:
   - Apply `StandardScaler` or `MinMaxScaler` for models sensitive to scale (e.g., regression)
4. Split data:
   - Train-test split (80/20) using `train_test_split()`

---

### Step 3: Exploratory Data Analysis (EDA)

#### Descriptive Statistics

- Summarize fares by airline, source, destination, and season.
- Check for correlations among numerical features.

#### Visual Analysis

- Plot distributions of fares, base fares, and taxes.
- Use boxplots to show fare variation across airlines.
- Plot average fare by month or season.
- Correlation heatmap to detect multicollinearity.

#### KPI Exploration

- Average fare per airline
- Most popular route (highest flight frequency)
- Seasonal fare variation (e.g., Eid, winter)
- Top 5 most expensive routes

---

### Step 4: Model Development (Baseline)

1. **Model Selection** — Start with Linear Regression as a baseline.
2. **Implementation**
   - Train a linear regression model using scikit-learn.
   - Evaluate using:
     - **R²** — Coefficient of determination
     - **MAE** — Mean Absolute Error
     - **RMSE** — Root Mean Squared Error
3. **Interpretation**
   - Visualize actual vs. predicted values.
   - Analyze residuals to detect underfitting or overfitting.
   - Document key findings.

---

### Step 5: Advanced Modeling & Optimization

1. **Try Multiple Models**
   - Ridge Regression
   - Lasso Regression
   - Decision Tree Regressor
   - Random Forest Regressor
   - Gradient Boosted Trees *(optional advanced step)*

2. **Model Tuning**
   - Use `GridSearchCV` or `RandomizedSearchCV` to tune hyperparameters.
   - Compare models using cross-validation results.

3. **Model Evaluation** — Summarize results in a comparison table:

   | Model | R² | MAE | RMSE | Notes |
   |---|---|---|---|---|
   | Linear Regression | | | | |
   | Ridge Regression | | | | |
   | Lasso Regression | | | | |
   | Decision Tree | | | | |
   | Random Forest | | | | |
   | Gradient Boosted Trees | | | | |

   Identify the best model balancing accuracy and simplicity.

4. **Regularization & Bias-Variance**
   - Demonstrate how Ridge and Lasso affect overfitting.
   - Plot the bias-variance tradeoff.

---

### Step 6: Model Interpretation & Insights

1. **Feature Importance**
   - For linear models: Examine coefficients.
   - For tree-based models: Plot feature importances.

2. **Insights**
   - What factors most influence fare prices?
   - How do airlines differ in pricing strategy?
   - Do certain seasons or routes consistently show higher fares?

3. **Communication**
   - Summarize results for non-technical stakeholders.
   - Include clear, data-backed recommendations.

---

## Suggested Visualizations

- Average Fare by Airline *(bar chart)*
- Total Fare Distribution *(histogram)*
- Fare Variation Across Seasons *(boxplot)*
- Feature Correlation Heatmap
- Feature Importance Plot *(Random Forest or Linear Regression)*
- Predicted vs. Actual Fares *(scatter plot)*

---

## Evaluation Criteria

Your project will be assessed based on:

- Correct implementation of each project step
- Code readability and documentation
- Quality of EDA and feature engineering
- Model performance and comparison rigor
- Clarity of insights and visualization quality
- Structure and completeness of deliverables

---

## Stretch Challenges

- Integrate the model into a **Flask** or **Streamlit** app for live predictions.
- Connect this project to your previous **Airflow pipeline** for scheduled retraining.
- Deploy the model locally as a simple **REST API**.

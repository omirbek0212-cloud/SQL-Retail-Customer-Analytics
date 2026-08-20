# 📊 Retail Customer Analytics & Cohort SQL Study

SQL-based customer analytics project focused on transactional behavior, customer retention, demographics, and spending dynamics over a 12-month period.

---

## 📌 Project Overview
This project analyzes customer transactions for a retail business over a 12-month period from June 2015 to May 2016.
The analysis combines transactional and customer demographic data to identify customer activity patterns, measure monthly business performance, and understand how spending behavior differs across customer segments.

The project was developed using SQLite and Python, with SQL used as the primary analytical tool.

---

## 🎯 Business Objectives
The analysis addresses four key business questions:

1. **Customer Retention & Cohort Analysis**
   * Which customers were continuously active during all 12 months?
   * What was their average transaction value?
   * How much did these customers spend per month on average?
2. **Monthly Operational Performance**
   * How does transaction activity change month by month?
   * What percentage of annual transactions and revenue does each month contribute?
   * How many active customers are served each month?
3. **Gender Demographic Analysis**
   * What proportion of active customers belongs to each gender?
   * How is monthly spending distributed across gender segments?
   * Are customer proportions consistent with their contribution to revenue?
4. **Age & Quarterly Analysis**
   * How are customers distributed across 10-year age groups?
   * Which age groups generate the most revenue?
   * How do customer spending patterns change across fiscal quarters?

---

## 🛠️ Tech Stack & SQL Techniques

| Technology | Purpose |
| :--- | :--- |
| **Python 3.10+** | Data preparation and analysis environment |
| **SQLite** | Relational database and SQL analytics |
| **Pandas** | Data ingestion, transformation, and validation |
| **Jupyter Notebook** | Interactive development and analysis |

### Key SQL Techniques
* Common Table Expressions (`WITH`)
* Aggregations (`SUM`, `AVG`, `COUNT`)
* Conditional aggregation with `CASE WHEN`
* Date manipulation with `strftime()`
* Cohort & Demographic segmentation
* Percentage and share calculations (`CROSS JOIN`)

---

## 🗂️ Dataset Structure
The analysis uses two primary tables:

### `transactions`
Contains individual customer transactions.

| Column | Description |
| :--- | :--- |
| `ID_client` | Unique customer identifier |
| `TRANSACTION_VALUE` | Transaction date/time |
| `Sum_m` | Transaction amount |

### `customers`
Contains customer-level demographic information.

| Column | Description |
| :--- | :--- |
| `Id_client` | Unique customer identifier |
| `Gender` | Customer gender |
| `Age` | Customer age |

---

## 📐 Key Analytical Tasks & SQL Implementation

### 1. Continuous Activity Cohort Analysis

#### Business Question
Which customers made transactions in every month of the 12-month observation period?

For each continuously active customer, the analysis calculates:
* Number of active months
* Total number of transactions
* Average transaction value
* Average monthly spending

```sql
WITH monthly_act AS (
    SELECT 
        ID_client,
        COUNT(DISTINCT strftime('%Y-%m', TRANSACTION_VALUE)) AS active_months,
        COUNT(*) AS total_ops,
        AVG(Sum_m) AS avg_check,
        SUM(Sum_m) / 12.0 AS avg_monthly_spend
    FROM transactions
    WHERE TRANSACTION_VALUE >= '2015-06-01'
      AND TRANSACTION_VALUE < '2016-06-01'
    GROUP BY ID_client
    HAVING active_months = 12
)
SELECT 
    c.Id_client,
    ROUND(m.avg_check, 2) AS avg_check_period,
    ROUND(m.avg_monthly_spend, 2) AS avg_monthly_spend,
    m.total_ops AS total_ops_period
FROM monthly_act m
JOIN customers c 
    ON m.ID_client = c.Id_client;
```
### 2. Monthly Dynamics & Annual Revenue Shares
Business Question
## The following metrics are calculated:

Average transaction value

Transactions per active customer
* Average transaction value
* Transactions per active customer
* Number of active customers
* Share of annual transaction volume
```sql
WITH monthly AS (
    SELECT 
        strftime('%Y-%m', TRANSACTION_VALUE) AS month_yr,
        COUNT(*) AS month_ops,
        SUM(Sum_m) AS month_sum,
        AVG(Sum_m) AS avg_check_month,
        COUNT(DISTINCT ID_client) AS active_clients
    FROM transactions
    WHERE TRANSACTION_VALUE >= '2015-06-01'
      AND TRANSACTION_VALUE < '2016-06-01'
    GROUP BY strftime('%Y-%m', TRANSACTION_VALUE)
),
totals AS (
    SELECT 
        COUNT(*) AS year_ops,
        SUM(Sum_m) AS year_sum
    FROM transactions
    WHERE TRANSACTION_VALUE >= '2015-06-01'
      AND TRANSACTION_VALUE < '2016-06-01'
)
SELECT 
    m.month_yr AS Month,
    ROUND(m.avg_check_month, 2) AS Avg_Check,
    ROUND(CAST(m.month_ops AS FLOAT) / m.active_clients, 2) AS Ops_Per_Client,
    m.active_clients AS Active_Clients,
    ROUND(m.month_ops * 100.0 / t.year_ops, 2) AS Ops_Share_Pct,
    ROUND(m.month_sum * 100.0 / t.year_sum, 2) AS Revenue_Share_Pct
FROM monthly m
CROSS JOIN totals t
ORDER BY m.month_yr;
```
### 3. Gender Demographic Breakdown

#### Business Question
How does customer activity and spending differ by gender?

This analysis combines transaction data with customer demographic information to measure monthly customer distribution and spending shares by gender.

```sql
SELECT 
    strftime('%Y-%m', t.TRANSACTION_VALUE) AS month_yr,
    ROUND(
        COUNT(DISTINCT CASE WHEN UPPER(c.Gender) = 'M' THEN t.ID_client END) * 100.0 / COUNT(DISTINCT t.ID_client), 
        2
    ) AS pct_clients_M,
    ROUND(
        COUNT(DISTINCT CASE WHEN UPPER(c.Gender) = 'F' THEN t.ID_client END) * 100.0 / COUNT(DISTINCT t.ID_client), 
        2
    ) AS pct_clients_F,
    ROUND(
        SUM(CASE WHEN UPPER(c.Gender) = 'M' THEN t.Sum_m ELSE 0 END) * 100.0 / SUM(t.Sum_m), 
        2
    ) AS spend_share_M,
    ROUND(
        SUM(CASE WHEN UPPER(c.Gender) = 'F' THEN t.Sum_m ELSE 0 END) * 100.0 / SUM(t.Sum_m), 
        2
    ) AS spend_share_F
FROM transactions t
LEFT JOIN customers c 
    ON t.ID_client = c.Id_client
WHERE t.TRANSACTION_VALUE >= '2015-06-01'
  AND t.TRANSACTION_VALUE < '2016-06-01'
GROUP BY strftime('%Y-%m', t.TRANSACTION_VALUE)
```
### 4. Age Group Stratification & Quarterly Trends

#### Business Question
How does customer spending and transaction activity differ across age groups, and how does this performance change between quarters?

Customers are dynamically grouped into 10-year brackets (`10-19`, `20-29`, `30-39`, etc.) while missing values are tagged as `NA`. Performance is aggregated both overall (`ALL_PERIOD`) and by fiscal quarter.

```sql
WITH age_prep AS (
    SELECT
        c.Id_client,
        CASE
            WHEN c.Age IS NULL OR c.Age = '' THEN 'NA'
            ELSE CAST((CAST(c.Age AS INT) / 10) * 10 AS TEXT) || '-' || CAST((CAST(c.Age AS INT) / 10) * 10 + 9 AS TEXT)
        END AS age_group,
        t.Sum_m,
        strftime('%Y', t.TRANSACTION_VALUE) || '-Q' || ((CAST(strftime('%m', t.TRANSACTION_VALUE) AS INT) + 2) / 3) AS quarter
    FROM transactions t
    LEFT JOIN customers c
        ON t.ID_client = c.Id_client
    WHERE t.TRANSACTION_VALUE >= '2015-06-01'
      AND t.TRANSACTION_VALUE < '2016-06-01'
)
SELECT
    age_group,
    'ALL_PERIOD' AS period,
    ROUND(SUM(Sum_m), 2) AS total_sum,
    COUNT(*) AS total_ops,
    ROUND(AVG(Sum_m), 2) AS avg_check
FROM age_prep
GROUP BY age_group

UNION ALL

SELECT
    age_group,
    quarter AS period,
    ROUND(SUM(Sum_m), 2) AS total_sum,
    COUNT(*) AS total_ops,
    ROUND(AVG(Sum_m), 2) AS avg_check
FROM age_prep
GROUP BY age_group, quarter

ORDER BY age_group, period;
```

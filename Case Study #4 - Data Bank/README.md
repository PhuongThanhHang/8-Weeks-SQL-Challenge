# Case Study #4 - Data Bank

---

## 🗂️ Table of Contents
- [Business Context](#business-context)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Case Study Questions and Solutions](#case-study-questions-and-solutions)

---

## Business Context

There is a new innovation in the financial industry called Neo-Banks: new aged digital only banks without physical branches.

Danny thought that there should be some sort of intersection between these new age banks, cryptocurrency and the data world…so he decides to launch a new initiative - Data Bank!

Data Bank runs just like any other digital bank - but it isn’t only for banking activities, they also have the world’s most secure distributed data storage platform!

Customers are allocated cloud data storage limits which are directly linked to how much money they have in their accounts. There are a few interesting caveats that go with this business model, and this is where the Data Bank team need your help!

The management team at Data Bank want to increase their total customer base - but also need some help tracking just how much data storage their customers will need.

---

## Entity Relationship Diagram

Below is the **Entity Relationship Diagram (ERD)** illustrating Data Bank’s database structure. For this case study there are 3 tables: `regions`, `customer_nodes`, `customer_transactions`

<p align="center">
  <img width="1069" height="464" alt="image" src="https://github.com/user-attachments/assets/1e4ccfb0-f022-4fe6-8d7f-15bc61d66f59" />
</p>

<p align="center"><em>Figure 1: Entity Relationship Diagram (ERD) of Data Bank database</em></p>


---

## Case Study Questions and Solutions

The analysis in this case study was performed with BigQuery as the main querying tool.

### A. Customer Nodes Exploration

**1. How many unique nodes are there on the Data Bank system?**
```sql
SELECT 
  COUNT (DISTINCT node_id) as unique_nodes
FROM `weeksqlchallenge-474608.4databank.customer_nodes`
```

**Answer**: There are 5 unique nodes on the Data Bank system

<img width="140" height="63" alt="image" src="https://github.com/user-attachments/assets/4b780df8-71b5-4aa5-95cf-88b410038756" />

**2. What is the number of nodes per region?**
```sql
SELECT 
  n.region_id,
  r.region_name,
  COUNT (DISTINCT node_id) as unique_nodes
FROM `weeksqlchallenge-474608.4databank.customer_nodes` as n
JOIN `weeksqlchallenge-474608.4databank.regions` as r
ON n.region_id = r.region_id
GROUP BY n.region_id, r.region_name
ORDER BY n.region_id
```

**Answer**:

<img width="361" height="180" alt="image" src="https://github.com/user-attachments/assets/6a96de94-c1d0-4c88-8d3f-f6a478a611ac" />

**3. How many customers are allocated to each region?**
```sql
SELECT 
  n.region_id,
  r.region_name,
  COUNT (DISTINCT customer_id) as total_customers
FROM `weeksqlchallenge-474608.4databank.customer_nodes` as n
JOIN `weeksqlchallenge-474608.4databank.regions` as r
ON n.region_id = r.region_id
GROUP BY n.region_id, r.region_name
ORDER BY n.region_id
```

**Answer**:

<img width="381" height="181" alt="image" src="https://github.com/user-attachments/assets/d8bd96a1-7314-436d-84b9-193bb02b570a" />

**4. How many days on average are customers reallocated to a different node?**
```sql
SELECT
  ROUND (AVG (DATE_DIFF (end_date,start_date, DAY)),1) as avg_days
FROM  `weeksqlchallenge-474608.4databank.customer_nodes`
WHERE end_date != '9999-12-31'
```

**Answer**: Customers are reassigned to a different node after roughly 14–15 days on average.

<img width="140" height="64" alt="image" src="https://github.com/user-attachments/assets/5f7ed545-f46e-4cb8-94c4-2d594d234f71" />

**5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?**

```sql
WITH regions AS (
  SELECT 
    n.region_id,
    r.region_name,
    DATE_DIFF(end_date, start_date, DAY) AS reallocation_days
  FROM `weeksqlchallenge-474608.4databank.customer_nodes` AS n
  JOIN `weeksqlchallenge-474608.4databank.regions` AS r
    ON n.region_id = r.region_id
  WHERE end_date != '9999-12-31'
)

SELECT
  region_id,
  region_name,
  APPROX_QUANTILES(reallocation_days, 100)[OFFSET(50)] AS median_days,
  APPROX_QUANTILES(reallocation_days, 100)[OFFSET(80)] AS p80_days,
  APPROX_QUANTILES(reallocation_days, 100)[OFFSET(95)] AS p95_days
FROM regions
GROUP BY region_id, region_name
ORDER BY region_id
```

**Answer**: Across all regions, customers remain on a given data node for 15 days on median. The 80th and 95th percentiles are consistently around 23–24 and 28 days across all regions, indicating a highly standardized load-balancing policy with minimal regional variation.
This consistency suggests the Data Bank infrastructure is well-optimized, stable, and evenly distributed, without region-specific bottlenecks or outlier behaviour.

<img width="576" height="180" alt="image" src="https://github.com/user-attachments/assets/5bdee9ae-0a1d-4817-9dbd-c23a02cd1a78" />

### B. Customer Transactions

**1. What is the unique count and total amount for each transaction type?**
```sql
SELECT
  txn_type,
  COUNT (txn_type) as transactions_count,
  SUM (txn_amount) as total_amount
FROM  `weeksqlchallenge-474608.4databank.customer_transactions`
GROUP BY txn_type
ORDER BY transactions_count DESC
```

**Answer**: Among all transaction types, deposits are performed most frequently, followed by purchases and withdrawals.

<img width="409" height="120" alt="image" src="https://github.com/user-attachments/assets/88bbb78a-3149-4fe1-ac68-9b0a86886aca" />

**2. What is the average total historical deposit counts and amounts for all customers?**
```sql
WITH deposit AS (
SELECT
  customer_id,
  COUNT (txn_type) as total_deposit_counts,
  SUM (txn_amount) as total_deposit_amount
FROM  `weeksqlchallenge-474608.4databank.customer_transactions`
where txn_type = 'deposit'
GROUP BY customer_id
ORDER BY customer_id
)

SELECT
  ROUND (AVG (total_deposit_counts),2) as avg_count,
  ROUND (AVG (total_deposit_amount),2) as avg_amount
FROM depositD
```

**Answer**: On average, each customer performs around 5 deposit transactions, with a total deposit value of approximately 2,718.34 USD per customer.

<img width="278" height="63" alt="image" src="https://github.com/user-attachments/assets/727f2561-9251-4837-a764-e34498acc3e2" />

**3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**
```sql
---For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**

WITH cte1 AS (
  SELECT
    customer_id,
    EXTRACT (MONTH from txn_date) as month,
    COUNT (CASE WHEN txn_type = 'deposit' THEN txn_type END) as deposits,
    COUNT (CASE WHEN txn_type = 'purchase' THEN txn_type END) as purchases, 
    COUNT (CASE WHEN txn_type = 'withdrawal' THEN txn_type END) as withdrawals
  FROM `weeksqlchallenge-474608.4databank.customer_transactions`
  GROUP BY customer_id, month
  ORDER BY customer_id, month
)

SELECT 
  month,
  COUNT (customer_id) as total_customer
FROM cte1
WHERE deposits >1 AND (purchases >= 1 OR withdrawals >= 1)
GROUP BY month
ORDER BY month
```

**Answer**:

<img width="275" height="150" alt="image" src="https://github.com/user-attachments/assets/a3d8a848-fc12-4af0-af0b-25964e827507" />

**4. What is the closing balance for each customer at the end of the month?**

**Idea**: 
I will generate a results table containing four columns:
- customer_id
- month: extract month from transaction date
- total_amount: the net transaction amount for each customer in each month (including both deposits and spending)
- closing_amount: the ending balance for that month, calculated by taking the previous month’s closing balance and subtracting the current month’s total_amount

**Note 1**: To calculate the total_amount for each month, I apply the following logic:
- Deposits increase the balance so treated as positive values.
- Purchases and withdrawals reduce the balance so treated as negative values. 
- So that I have to convert deposit, withdrawal, and purchase values into +txn_amount or –txn_amount, by using
```sql
SUM(CASE 
        WHEN txn_type = 'deposit' THEN txn_amount
        WHEN txn_type IN ('withdrawal', 'purchase') THEN -txn_amount
    END) AS total_amount
```
OR
```sql
SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount
         ELSE -txn_amount END) AS total_amount
```
**Note 2**: To calculate the closing_amount for each month, I apply the following logic `closing month N = closing month N-1 + total_amount month N`
and use `SUM(...) OVER (PARTITION BY ... ORDER BY ...)`:
- SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY month): calculate total_amount for each month, grouped by each customer.
- After ORDER BY month, I Add an additional condition to control the sorting `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`: It means take all rows from the beginning (UNBOUNDED PRECEDING) up to the current row (CURRENT ROW).
- So that `SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` means calculate the running total of total_amount month by month for each customer, where the cumulative value stops at the current month.
- Example: 

| Month | Total Amount | Closing Balance |
|-------|--------------|-----------------|
| Jan   | +60          | 60              |
| Feb   | +50          | 60 + 50 = 110   |
| Mar   | -20          | 110 - 20 = 90   |

```sql
WITH cte1 AS (
  SELECT 
    customer_id,
    EXTRACT (MONTH from txn_date) as month,
    SUM ((CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -1*txn_amount END)) AS total_amount
  FROM `weeksqlchallenge-474608.4databank.customer_transactions`
  GROUP BY customer_id, month
  ORDER BY customer_id, month
)

SELECT 
  customer_id,
  month,
  total_amount,
  SUM (total_amount) OVER (PARTITION BY customer_id ORDER BY month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as closing_amount
FROM cte1
ORDER BY customer_id, month
```

**Answer**: Here are the first 10 rows of the result table, which contains a total of 1,720 records.

<img width="493" height="353" alt="image" src="https://github.com/user-attachments/assets/7d9205d7-9e5b-4674-92d9-d61acb531788" />

**5. What is the percentage of customers who increase their closing balance by more than 5%?**
```sql
WITH cte1 AS (
  SELECT 
    customer_id,
    EXTRACT (MONTH from txn_date) as month,
    SUM ((CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -1*txn_amount END)) AS total_amount
  FROM `weeksqlchallenge-474608.4databank.customer_transactions`
  GROUP BY customer_id, month
  ORDER BY customer_id, month
),

closing_balance AS (
  SELECT 
    customer_id,
    month,
    total_amount,
    SUM (total_amount) OVER (PARTITION BY customer_id ORDER BY month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as closing_amount
  FROM cte1
ORDER BY customer_id, month
),

percentage AS (
  SELECT
    customer_id,
    month,
    total_amount,
    closing_amount,
    LAG(closing_amount) OVER (PARTITION BY customer_id ORDER BY month) AS previous_closing_amount,
    closing_amount - LAG(closing_amount) OVER (PARTITION BY customer_id ORDER BY month) AS change_amount,
    ROUND(
  SAFE_DIVIDE(
    (closing_amount - LAG(closing_amount) OVER (PARTITION BY customer_id ORDER BY month)),
    LAG(closing_amount) OVER (PARTITION BY customer_id ORDER BY month)
  ) * 100,
  2
) AS percent
  FROM closing_balance
  ORDER BY customer_id, month
)

SELECT 
COUNT (DISTINCT CASE WHEN percent > 5 THEN customer_id END) / COUNT (DISTINCT customer_id) *100 as percentage_customer
FROM percentage 
```

**Answer**: 75.8% of customers increased their closing balance by more than 5%.

<img width="175" height="57" alt="image" src="https://github.com/user-attachments/assets/74ad013c-1eca-4721-a6be-6b45d3a9f2cf" />

### C. Data Allocation Challenge

### D. Extra Challenge


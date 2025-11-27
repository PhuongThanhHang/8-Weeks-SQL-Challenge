# Case Study #8 - Fresh Segments

---

## 🗂️ Table of Contents
- [Business Context](#business-context)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Case Study Questions and Solutions](#case-study-questions-and-solutions)

---

## Business Context

Danny created Fresh Segments, a digital marketing agency that helps other businesses analyse trends in online ad click behaviour for their unique customer base.

Clients share their customer lists with the Fresh Segments team who then aggregate interest metrics and generate a single dataset worth of metrics for further analysis.

In particular - the composition and rankings for different interests are provided for each client showing the proportion of their customer list who interacted with online assets related to each interest for each month.

Danny has asked for your assistance to analyse aggregated metrics for an example client and provide some high level insights about the customer list and their interests.
---

## Entity Relationship Diagram

Below is the **Entity Relationship Diagram (ERD)** illustrating Fresh Segments’s database structure. For this case study there are 2 tables: `interest_metrics`, `interest_map`

---

## Case Study Questions and Solutions

The analysis in this case study was performed with BigQuery as the main querying tool.

### A. Data Exploration and Cleansing

**1. Update the fresh_segments.interest_metrics table by modifying the month_year column to be a date data type with the start of the month?**

- If you are using a paid BigQuery plan, you can directly update the table using the query below.

```sql
UPDATE weeksqlchallenge-474608.8freshsegments.interest_metrics 
SET month_year = DATE( 
  CAST(SUBSTR(month_year, 4, 4) AS INT64), 
  CAST(SUBSTR(month_year, 1, 2) AS INT64), 1 ) 
WHERE month_year IS NOT NULL
```

- If you are using the free version of BigQuery, which does not support UPDATE, DELETE, INSERT, MERGE, or ALTER TABLE, you should create a new table based on the existing one and then deprecate the old table.
If you only need to modify the month_year column and do not need to keep the original table, you can use CREATE OR REPLACE TABLE to overwrite it directly.

```sql
CREATE OR REPLACE TABLE `weeksqlchallenge-474608.8freshsegments.interest_metrics` AS
SELECT
  month,
  year,
  DATE(
      CAST(SUBSTR(month_year, 4, 4) AS INT64), 
      CAST(SUBSTR(month_year, 1, 2) AS INT64), 
      1
  ) AS month_year,
  interest_id,
  composition,
  index_value,
  ranking,
  percentile_ranking
FROM `weeksqlchallenge-474608.8freshsegments.interest_metrics`
```
**Answer**:

<img width="623" height="111" alt="image" src="https://github.com/user-attachments/assets/f34ce3b0-8b5b-47e9-8a8f-d9e27023a461" />

**2. What is count of records in the fresh_segments.interest_metrics for each month_year value sorted in chronological order (earliest to latest) with the null values appearing first?**

```sql
SELECT
  month_year,
  COUNT (*) as count_
FROM `weeksqlchallenge-474608.8freshsegments.interest_metrics`
GROUP BY month_year
ORDER BY month_year NULLS FIRST
```

**Answer**:


**3. What do you think we should do with these null values in the fresh_segments.interest_metrics**

- For month, year, and month_year, NULL values indicate missing time information, so they should be kept as NULL but excluded from trend and time-series analyses.

- For interest_id, NULL means it cannot be mapped to any segment, so the row should be excluded.

- For numeric columns, NULL values are very rare and should be kept as NULL since zero is not equivalent to NULL.

**4. How many interest_id values exist in the fresh_segments.interest_metrics table but not in the fresh_segments.interest_map table? What about the other way around?**

```sql

```

**Answer**:


**5. Summarise the id values in the fresh_segments.interest_map by its total record count in this table**

```sql

```

**Answer**:


**6. What sort of table join should we perform for our analysis and why? Check your logic by checking the rows where interest_id = 21246 in your joined output and include all columns from fresh_segments.interest_metrics and all columns from fresh_segments.interest_map except from the id column.**

```sql

```

**Answer**:


**7. Are there any records in your joined table where the month_year value is before the created_at value from the fresh_segments.interest_map table? Do you think these values are valid and why?**

```sql

```

**Answer**:


### B. Interest Analysis

### C. Segment Analysis

### D.Index Analysis

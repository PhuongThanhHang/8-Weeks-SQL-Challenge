# Case Study #5 - Data Mart

---

## 🗂️ Table of Contents
- [Business Context](#business-context)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Case Study Questions and Solutions](#case-study-questions-and-solutions)

---

## Business Context

Data Mart is Danny’s latest venture and after running international operations for his online supermarket that specialises in fresh produce - Danny is asking for your support to analyse his sales performance.

In June 2020 - large scale supply changes were made at Data Mart. All Data Mart products now use sustainable packaging methods in every single step from the farm all the way to the customer.

The key business question he wants you to help him answer are the following:

Your objective as a Data Analyst is to answer these questions:
- What was the quantifiable impact of the changes introduced in June 2020?
- Which platform, region, segment and customer types were the most impacted by this change?
- What can we do about future introduction of similar sustainability updates to the business to minimise impact on sales?

---

## Entity Relationship Diagram

Below is the **Entity Relationship Diagram (ERD)** illustrating Data Mart’s database structure. For this case study there is only a single table: `weekly_sales`

<p align="center">
  <img width="420" alt="ERD - Data Mart" src="https://github.com/user-attachments/assets/94f47f46-af86-4046-91f5-f0dd62e21b29" />
</p>

<p align="center"><em>Figure 1: Entity Relationship Diagram (ERD) of Data Mart database</em></p>

**Column Dictionary**
- `week_date`: is represents the start of the sales week.
- `region`: the business region of Data Mart, reflecting market characteristics by area.
- `platform`: the sales channel, gồm Retail (cửa hàng) và Online (Shopify).
- `segment` and `customer_type`: is represents personal age and demographics information that is shared with Data Mart.
- `transactions`: is the count of unique purchases made through Data Mart
- `sales`: is the actual dollar amount of purchases.

**Example Rows**

10 random rows are shown in the table output below from table `weekly_sales`

<p align="center">
  <img width="649" height="356" alt="131438417-1e21efa3-9924-490f-9bff-3c28cce41a37" src="https://github.com/user-attachments/assets/34ae83e8-a9ca-4285-bc93-89abfd1ac27b" />
<p align="center"><em>Figure 2: Example rows of table weekly_sales</em></p>

---

## Case Study Questions and Solutions

The analysis in this case study was performed with BigQuery as the main querying tool.

### A. Data Cleansing Steps

**Questions**

In a single query, perform the following operations and generate a new table in the data_mart schema named `clean_weekly_sales`:
- Convert the `week_date` to a DATE format
- Add a `week_number` as the second column for each `week_date` value, for example any value from the 1st of January to 7th of January will be 1, 8th to 14th will be 2 etc
- Add a `month_number` with the calendar month for each `week_date` value as the 3rd column
- Add a `calendar_year` column as the 4th column containing either 2018, 2019 or 2020 values
- Add a new column called `age_band` after the original `segment` column using the following mapping on the number inside the `segment` value
- Add a new `demographic` column using the following mapping for the first letter in the `segment` values:
- Ensure all null string values with an "unknown" string value in the original `segment` column as well as the new `age_band` and `demographic` columns
- Generate a new `avg_transaction` column as the sales value divided by transactions rounded to 2 decimal places for each record

**Answer**
Generate a new table in the data_mart schema named `clean_weekly_sales`:
| Column Name | Implementation Steps |
|--------------|-------------|
| week_date | Unchanged |
| week_number | Extract number of week using EXTRACT ISOWEEK |
| month_number | Extract number of week using EXTRACT MONTH |
| calender_year | Extract number of week using EXTRACT YEAR |
| region | Unchanged |
| platform | Unchanged |
| segment | Unchanged |
| age_band | Use the CASE statement with conditional logic on segment, applying LIKE '%1%' = Young Adults, LIKE '%2%' = Middle Aged, LIKE '%3%' OR '%4%' = Retirees, and NULL = Unknown. |
| demographic | Use the CASE statement with conditional logic on segment, applying LIKE 'C%' = Couples, LIKE 'F%' = Families, and NULL = Unknown. |
| customer_type | Unchanged |
| transactions | Unchanged |
| avg_transaction | Divide sales with transactions and round up to 2 decimal places |
| sales | Unchanged |

```sql
CREATE TABLE `weeksqlchallenge-474608.5datamart.clean_weekly_sales` AS
SELECT
week_date,
extract (ISOWEEK from week_date) as week_number,
extract (MONTH from week_date) as month_number,
extract (YEAR from week_date) as calender_year,
region,
platform,
segment,
Case
  When segment LIKE '%1' then 'Young Adults'
  When segment LIKE '%2' then 'Middle Aged'
  When segment LIKE '%3' OR segment LIKE '%4' then 'Retirees'
  When segment = 'null' then 'unknown'
End as age_band,
Case
  When segment LIKE 'C%' then 'Couples'
  When segment LIKE 'F%' then 'Families'
  When segment = 'null' then 'unknown'
End as demographic,
customer_type,
transactions,
round(safe_divide(sales, transactions), 2) AS avg_transaction,
sales
FROM `weeksqlchallenge-474608.5datamart.weekly_sales`
```
<p align="center">
  <img width="1769" height="313" alt="AScreenshot 2025-11-06 234119" src="https://github.com/user-attachments/assets/2f815a36-8db7-4c2d-b2d5-a0641244d3c5" />
</p>

### B. Data Exploration

**1. What day of the week is used for each week_date value?**
```sql
SELECT
 DISTINCT FORMAT_DATE('%A', week_date) AS day_of_week_name,
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`
```

**Answer**: Monday is used for each week_date value

<img width="304" height="86" alt="image" src="https://github.com/user-attachments/assets/c17210ca-7836-42b0-b3df-98dfe25cf8fe" />

**2. What range of week numbers are missing from the dataset?**
- Create a reference table for week numbers by unnesting an array from 1 to 53, which represents the total number of weeks in a year according to the ISO week standard.
- Perform a LEFT JOIN between the clean_weekly_sales table and the week reference table to identify the weeks that are missing from the sales data.

```sql
WITH week_reference AS
  (SELECT *
  FROM UNNEST (GENERATE_ARRAY(1,53)) AS week_number)
SELECT
  a.week_number as missing_week_number
FROM week_reference as a
  LEFT JOIN `weeksqlchallenge-474608.5datamart.clean_weekly_sales` as b
  ON a.week_number = b.week_number
  WHERE b.week_number IS NULL
```

**Answer**: The dataset contains a total of 29 missing week_number. The missing week number is 1,2,3,...,12,37,38,39,...53. The table below shows first 5 rows of the result table for the missing weeks.

<img width="168" height="161" alt="image" src="https://github.com/user-attachments/assets/e4ad8463-89d5-4e18-82cd-892791047ba4" />

**3. How many total transactions were there for each year in the dataset?**
```sql
SELECT
  calender_year as year,
  sum (transactions) as total_transactions
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`
GROUP BY calender_year
ORDER BY calender_year
```

**Answer**:

<img width="252" height="108" alt="image" src="https://github.com/user-attachments/assets/b1618c74-fe49-42cd-87f7-c9ebea9cce4b" />

**4. What is the total sales for each region for each month?**
```sql
SELECT
  month_number as month,
  region,
  sum (sales) as total_sales
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`
GROUP BY month_number, region
ORDER BY month_number, region
```

**Answer**: The table below shows first 5 rows of the result table for the total sales for each region by month.

<img width="452" height="165" alt="image" src="https://github.com/user-attachments/assets/4fc249f4-4215-4fd0-a084-84aaddff927a" />

**5. What is the total count of transactions for each platform?**
```sql
SELECT
  platform,
  sum (transactions) as total_transactions
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`
GROUP BY platform
ORDER BY platform
```

**Answer**: The results indicate that the retail platform records significantly higher total transactions than Shopify, suggesting that customers engage more frequently with in-store purchases.

<img width="330" height="83" alt="image" src="https://github.com/user-attachments/assets/be3194e7-d7b3-4ba2-8aad-a01e88947073" />

**6. What is the percentage of sales for Retail vs Shopify for each month?**
- Use SUM(sales) / SUM(SUM(sales)) OVER (...) to calculate each platform’s share of total monthly sales, where the inner SUM() aggregates by platform and the outer window SUM() computes the total across all platforms in that month.
- Apply PIVOT(MAX(percentage)) to transform the platform values into separate columns (Retail, Shopify) and display their respective percentages side by side.

```sql
WITH sales_table AS (
SELECT
  calender_year,
  month_number,
  platform,
  ROUND (SUM (sales)/ SUM (SUM(sales)) OVER (PARTITION BY calender_year, month_number) *100,2) AS percentage
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`
GROUP BY calender_year, month_number, platform
ORDER BY calender_year, month_number, platform)

SELECT
  calender_year as year,
  month_number as month,
  Retail as retail_percentage,
  Shopify as shopify_percentage
FROM sales_table
PIVOT (MAX(percentage) FOR platform IN ('Retail', 'Shopify'))
ORDER BY calender_year, month_number
```

**Answer**: The percentage of sales from the retail platform remains consistently above 96% each month, whereas the Shopify platform accounts for only around 2–3% of total sales.

<img width="497" height="164" alt="image" src="https://github.com/user-attachments/assets/c5fcd3be-60a3-4773-8a83-ce0bf7e2d90a" />

**7. What is the percentage of sales by demographic for each year in the dataset?**
- Use SUM(sales) / SUM(SUM(sales)) OVER (PARTITION BY calender_year) to calculate each demographic group’s percentage contribution to the total yearly sales — the inner SUM() aggregates by demographic, while the outer window SUM() totals sales across all demographics for that year.
- Apply MAX(CASE WHEN … THEN … END) to pivot demographic values into columns, taking the maximum value within each group since each demographic-year combination has only one percentage value.

```sql
WITH sales_table AS (
SELECT
  calender_year as year,
  demographic,
  SUM(sales) AS total_sales,
  ROUND (SUM(sales) / SUM(SUM(sales)) OVER (PARTITION BY calender_year)*100,2) AS percentage
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`
GROUP BY calender_year, demographic
ORDER BY calender_year, demographic)

SELECT
  year,
  MAX (CASE WHEN demographic = 'Couples' then percentage END) AS couples_percentage,
  MAX (CASE WHEN demographic = 'Families' then percentage END) AS families_percentage,
  MAX (CASE WHEN demographic = 'unknown' then percentage END) AS unknown_percentage
FROM sales_table
ORDER BY year
```

**Answer**: Excluding the unknown demographic, the proportion of family customers (around 31–32%) is higher than that of couple customers (approximately 26–28%). Although the difference is not substantial, from 2018 to 2020, the share of couple customers at Data Mart increased slightly by about 2%, while the proportion of family customers remained unchanged.

<img width="550" height="110" alt="image" src="https://github.com/user-attachments/assets/0c6a644d-536a-435e-af09-e595cd78a82f" />

**8. Which age_band and demographic values contribute the most to Retail sales?**
```sql
SELECT
  age_band,
  demographic,
  ROUND (SUM (sales)/ SUM (SUM(sales)) OVER () *100, 2) AS percentage
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`
WHERE platform = 'Retail'
GROUP BY age_band,demographic
ORDER BY percentage DESC
```

**Answer**: Retired customers, both families and couples, are the top contributors to total retail sales, accounting for 16.73% and 16.07%, respectively. Additionally, the middle-aged family segment also contributes 11% of total revenue.

<img width="530" height="219" alt="image" src="https://github.com/user-attachments/assets/7c9c868b-c8c5-4a1c-a80d-8d8aff5bd411" />

**9. Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?**
```sql
WITH avg_transactions AS (
SELECT
  calender_year,
  platform,
  ROUND (SUM (sales)/ SUM (transactions), 2) AS avg_transactions_size
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`
GROUP BY calender_year, platform
ORDER BY calender_year, platform )

SELECT
  calender_year as year,
  Retail as retail_transactions_size,
  Shopify as shopify_transactions_size
FROM avg_transactions
PIVOT (MAX(avg_transactions_size) FOR platform IN ('Retail', 'Shopify'))
ORDER BY year
```

**Answer**: You should not use the avg_transaction column directly to calculate yearly averages, as it represents the average per smaller aggregation groups (e.g., weekly data). To obtain the accurate average transaction value per year, you need to compute it as **SUM(sales) / SUM(transactions)** for each year and platform.

<img width="472" height="107" alt="image" src="https://github.com/user-attachments/assets/fa030682-e3e6-4187-89f8-d5d0ceedb7eb" />

### C. Before & After Analysis

Taking the week_date value of 2020-06-15 as the baseline week where the Data Mart sustainable packaging changes came into effect. We would include all week_date values for 2020-06-15 as the start of the period after the change and the previous week_date values would be before

Using this analysis approach - answer the following questions:

**1. What is the total sales for the 4 weeks before and after 2020-06-15? What is the growth or reduction rate in actual values and percentage of sales?**
- DATE_SUB and DATE_ADD are used to define a 4-week window before and after June 15 2020, allowing comparison of total sales between the pre- and post-event periods.
- SUM(CASE…) calculates the total sales within the specified date range.
  
```sql
WITH sales_change AS (
SELECT
  SUM (Case when week_date >= DATE_SUB ('2020-06-15', INTERVAL 4 WEEK) AND week_date < DATE '2020-06-15' then sales END) AS before_sales,
  SUM (Case when week_date < DATE_ADD ('2020-06-15', INTERVAL 4 WEEK) AND week_date >= DATE '2020-06-15' then sales END) AS after_sales
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`)

SELECT
  before_sales,
  after_sales,
  after_sales - before_sales as sales_diff,
  ROUND ((after_sales - before_sales)/ before_sales *100, 2) as percentage
FROM sales_change
```

**Answer**: Since switching to the new sustainable packaging, our sales have fallen by $26.88 million (a 1.15% decline). This shows that new packaging can hurt sales if customers don't immediately recognize the product on the shelf or simply don't like the new design.

<img width="502" height="58" alt="image" src="https://github.com/user-attachments/assets/5d065e98-fae4-49f6-8397-8311c88a7439" />

**2. What about the entire 12 weeks before and after?**
```sql
WITH sales_change AS (
SELECT
  SUM (Case when week_date >= DATE_SUB ('2020-06-15', INTERVAL 12 WEEK) AND week_date < DATE '2020-06-15' then sales END) AS before_sales,
  SUM (Case when week_date < DATE_ADD ('2020-06-15', INTERVAL 12 WEEK) AND week_date >= DATE '2020-06-15' then sales END) AS after_sales
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`)

SELECT
  before_sales,
  after_sales,
  after_sales - before_sales as sales_diff,
  ROUND ((after_sales - before_sales)/ before_sales *100, 2) as percentage
FROM sales_change
```

**Answer**:

<img width="501" height="54" alt="image" src="https://github.com/user-attachments/assets/83f9d44d-b516-4773-8c17-a4716196853b" />

**3. How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?**
- Using DATE_SUB(PARSE_DATE(CONCAT(...))) to dynamically calculate the date 4 weeks before June 15 of each calender_year, ensuring the before_sales sum adjusts automatically for different years.
  
```sql
WITH sales_change AS (
SELECT
  calender_year as year,
  SUM (Case when week_date >= DATE_SUB (PARSE_DATE('%Y%m%d', CONCAT(calender_year, '06', '15')), INTERVAL 4 WEEK)
      AND week_date < PARSE_DATE('%Y%m%d', CONCAT(calender_year, '06', '15')) then sales END) AS before_sales,
  SUM (Case when week_date < DATE_ADD (PARSE_DATE('%Y%m%d', CONCAT(calender_year, '06', '15')), INTERVAL 4 WEEK)
      AND week_date >= PARSE_DATE('%Y%m%d', CONCAT(calender_year, '06', '15')) then sales END) AS after_sales
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`
GROUP BY calender_year)

SELECT
  year,
  before_sales,
  after_sales,
  after_sales - before_sales as sales_diff,
  ROUND ((after_sales - before_sales)/ before_sales *100, 2) as percentage
FROM sales_change
ORDER BY year
```

**Answer**: To analyze this further, we compared the sales performance for the four weeks before and after the benchmark date of June 15th across the years 2018, 2019, and 2020.
In both 2018 and 2019, sales for the four weeks following June 15th saw a slight increase of 0.1% to 0.2%. While these increases are minor, this stable, positive trend contrasts sharply with the 1.15% decline we observed in 2020.
This suggests that the change in packaging truly had a negative impact on sales.

<img width="626" height="110" alt="image" src="https://github.com/user-attachments/assets/f6432f2b-2b06-4aed-a04f-637680ec67b2" />

### D. Bonus Questions

**Questions**

Which areas of the business have the highest negative impact in sales metrics performance in 2020 for the 12 week before and after period?
- region
- platform
- age_band
- demographic
- customer_type
Do you have any further recommendations for Danny’s team at Data Mart or any interesting insights based off this analysis?

**Answer**
- This query calculates sales_change by region for the 12 weeks before and after the packaging change. The same query can be applied by replacing region with platform, age_band, demographic, or customer_type.

```sql
WITH sales_change AS (
SELECT
  region,
  SUM (Case when week_date >= DATE_SUB ('2020-06-15', INTERVAL 12 WEEK) AND week_date < DATE '2020-06-15' then sales END) AS before_sales,
  SUM (Case when week_date < DATE_ADD ('2020-06-15', INTERVAL 12 WEEK) AND week_date >= DATE '2020-06-15' then sales END) AS after_sales
FROM `weeksqlchallenge-474608.5datamart.clean_weekly_sales`
GROUP BY region)

SELECT
  region,
  before_sales,
  after_sales,
  after_sales - before_sales as sales_diff,
  ROUND ((after_sales - before_sales)/ before_sales *100, 2) as percentage
FROM sales_change
ORDER BY percentage DESC
```
- Region: In 2020, EU sales increased by 4.73%, while OCEANIA and ASIA saw the largest drops of 3.26%. However, EU only contributed 1.5% of total sales, which is minimal, whereas OCEANIA, ASIA, and AFRICA contributed significantly but experienced steep declines. Therefore, focusing on EU is not recommended.
- Shopify sales grew by 7%, while retail declined by 2.43%. However, since retail accounts for a much larger share of total sales (97% vs. 2–3% for Shopify), the company should focus on improving retail rather than investing heavily in Shopify.
- After the packaging change, sales decreased across all age bands and demographic groups.
- For the three customer_type segments, new customers increased by 1%, while existing and returning customers declined by 2–3%. The rise in new customers likely reflects their lack of prior experience with the old packaging, leading them to try the product, whereas existing and returning customers showed a drop.

**In summary**, the packaging change negatively impacted sales, so the company should revert to the old packaging and conduct customer surveys to understand their reactions to the new packaging.

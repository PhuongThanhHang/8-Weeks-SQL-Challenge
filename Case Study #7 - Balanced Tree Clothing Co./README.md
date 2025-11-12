# Case Study #7 - Balanced Tree Clothing Co.

---

## 🗂️ Table of Contents
- [Business Context](#business-context)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Case Study Questions and Solutions](#case-study-questions-and-solutions)

---

## Business Context

Balanced Tree Clothing Company prides themselves on providing an optimised range of clothing and lifestyle wear for the modern adventurer!

Danny, the CEO of this trendy fashion company has asked you to assist the team’s merchandising teams analyse their sales performance and generate a basic financial report to share with the wider business.

---

## Entity Relationship Diagram

Below is the **Entity Relationship Diagram (ERD)** illustrating Balanced Tree Clothing Company’s database structure. For this case study there is a total of 4 datasets

<p align="center">
  <img width="688" height="408" alt="image" src="https://github.com/user-attachments/assets/7e0afe51-2f44-4b27-b28b-2940ac6adf4b" />
</p>
<p align="center"><em>Figure 1: Entity Relationship Diagram (ERD) of Balanced Tree Clothing database</em></p>

---

## Case Study Questions and Solutions

The analysis in this case study was performed with BigQuery as the main querying tool.

### A. High Level Sales Analysis
**1. What was the total quantity sold for all products?**
```sql
—--all products
SELECT
  SUM(qty) AS total_quantity
FROM `weeksqlchallenge-474608.7balancedtree.sales`

--- each product
SELECT
  pd.product_name,
  SUM(qty) AS total_quantity
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.product_name
ORDER BY total_quantity DESC
```

**Answer**: There are 45,216 products sold, with women’s Jackets & Jeans and men’s Shirts being the top sellers.

<img width="388" height="355" alt="image" src="https://github.com/user-attachments/assets/3e81bed1-de24-4343-bf6f-ed1ea41f55b3" />


**2. What is the total generated revenue for all products before discounts?**
```sql
--all products
SELECT
  SUM(qty*price) AS total_quantity
FROM `weeksqlchallenge-474608.7balancedtree.sales`

--- each product
SELECT
  pd.product_name,
  SUM(s.qty * s.price) AS total_revenue
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.product_name
ORDER BY total_revenue DESC
```

**Answer**: The total revenue before discounts is 1,289,453 USD, and the revenue for each product is shown in the table below. 

<img width="384" height="353" alt="image" src="https://github.com/user-attachments/assets/d17151f0-e2be-44be-ae24-145f27ea2c29" />

**3. What was the total discount amount for all products?**
```sql
--all products
SELECT
  ROUND (SUM(qty*price*discount/100),2) AS total_discount
FROM `weeksqlchallenge-474608.7balancedtree.sales`

--- each product
SELECT
  pd.product_name,
  ROUND (SUM(s.qty*s.price*s.discount/100),2) AS total_discount
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.product_name
ORDER BY total_discount DESC
```

**Answer**: The total discount amount for all products is 156229.14 USD, and the total discount amount for each product is shown in the table below. 

<img width="385" height="353" alt="image" src="https://github.com/user-attachments/assets/e3e819a2-4c76-4110-9d3a-64610fdb22c7" />

### B. Transaction Analysis

**1. How many unique transactions were there?**
```sql
SELECT
  COUNT (DISTINCT txn_id) as unique_transactions
FROM `weeksqlchallenge-474608.7balancedtree.sales`
```

**Answer**: There were total 2500 unique transactions

<img width="155" height="51" alt="image" src="https://github.com/user-attachments/assets/3c4125ea-fda2-429b-8aa7-ca1facc01632" />

**2. What is the average unique products purchased in each transaction?**
```sql
WITH cte AS (
SELECT
  txn_id,
  COUNT (DISTINCT product_id) as unique_products
FROM `weeksqlchallenge-474608.7balancedtree.sales`
GROUP BY txn_id)
SELECT
  ROUND (AVG (unique_products),1) as avg_unique_products
FROM cte
```

**Answer**: On average, each transaction contains about six unique products.

<img width="164" height="57" alt="image" src="https://github.com/user-attachments/assets/3cfca440-b5db-4e5f-90dd-07866bfcfb9d" />

**3. What are the 25th, 50th and 75th percentile values for the revenue per transaction?**
```sql
WITH cte AS (
SELECT
  txn_id,
  SUM (qty * price) as revenue_per_transaction
FROM `weeksqlchallenge-474608.7balancedtree.sales`
GROUP BY txn_id)

SELECT
  qs[OFFSET(1)] AS p25,
  qs[OFFSET(2)] AS p50,
  qs[OFFSET(3)] AS p75
FROM (
  SELECT APPROX_QUANTILES(revenue_per_transaction, 4) AS qs
  FROM cte)
```

**Answer**:

<img width="379" height="58" alt="image" src="https://github.com/user-attachments/assets/7b060f47-4591-4676-ad62-6006aa432fee" />

**4. What is the average discount value per transaction?**
```sql
WITH cte AS (
SELECT
  txn_id,
  ROUND (SUM(qty * price * discount/100), 2) as discount_amount
FROM `weeksqlchallenge-474608.7balancedtree.sales`
GROUP BY txn_id)
SELECT
  ROUND (AVG (cte.discount_amount),2) as discount_per_transaction
FROM cte
```
**Answer**:

<img width="174" height="57" alt="image" src="https://github.com/user-attachments/assets/b7e44fe0-8c7a-4c3a-8b76-652cfac977d9" />

**5. What is the percentage split of all transactions for members vs non-members?**
```sql
SELECT
  member,
  COUNT (DISTINCT txn_id) as transactions,
  ROUND (COUNT (DISTINCT txn_id) / SUM (COUNT (DISTINCT txn_id)) OVER () *100,2) as percentage
FROM `weeksqlchallenge-474608.7balancedtree.sales`
GROUP BY member
```

**Answer**: Among all transactions from Balanced Tree, 60.2% were made by members, while the remaining 39.8% came from non-members.

<img width="346" height="83" alt="image" src="https://github.com/user-attachments/assets/115b9583-3ab0-4b62-9ea5-ee7fcbaf9eda" />

**6. What is the average revenue for member transactions and non-member transactions?**
```sql
SELECT
  member,
  ROUND (SUM (qty * price)/ COUNT (DISTINCT txn_id),2) as avg_revenue
FROM `weeksqlchallenge-474608.7balancedtree.sales`
GROUP BY member
```

**Answer**:

<img width="225" height="83" alt="image" src="https://github.com/user-attachments/assets/af9f9a72-e2d7-419a-96cf-88e2c74e1f37" />

### C. Product Analysis

**1. What are the top 3 products by total revenue before discount?**
```sql
SELECT
  pd.product_name,
  SUM (s.qty * s.price) as total_revenue
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.product_name
ORDER BY total_revenue DESC
LIMIT 3
```

**Answer**: The top three products by total revenue before discounts are the Blue Polo Shirt – Mens (217,683 USD), the Grey Fashion Jacket – Womens (209,304 USD), and the White Tee Shirt – Mens (152,000 USD).

<img width="246" height="83" alt="image" src="https://github.com/user-attachments/assets/82b3a1d9-94c9-49d4-95fc-f6c3cf77faad" />

**2. What is the total quantity, revenue and discount for each segment?**
```sql
SELECT
  pd.segment_name,
  SUM (s.qty) as total_quantity,
  SUM (s.qty * s.price) as total_revenue,
  ROUND (SUM (s.qty * s.price * s.discount/100),2) as total_discount
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.segment_name
```

**Answer**:

<img width="433" height="102" alt="image" src="https://github.com/user-attachments/assets/864834b6-da7e-4504-a782-435dfb72769b" />

**3. What is the top selling product for each segment?**
```sql
WITH cte AS (
SELECT
  pd.segment_name,
  pd.product_name,
  SUM (s.qty) as total_quantity,
  RANK () OVER (PARTITION BY pd.segment_name ORDER BY SUM (s.qty) DESC) as ranking
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.segment_name, pd.product_name
)
SELECT
  segment_name,
  product_name,
  total_quantity
FROM cte
WHERE ranking = 1
```

**Answer**: The table below shows the top selling product for each segment

<img width="396" height="103" alt="image" src="https://github.com/user-attachments/assets/5f92a661-4cc1-41fe-9295-038414764393" />

**4. What is the total quantity, revenue and discount for each category?**
```sql
SELECT
  pd.category_name,
  SUM (s.qty) as total_quantity,
  SUM (s.qty * s.price) as total_revenue,
  ROUND (SUM (s.qty * s.price * s.discount/100),2) as total_discount
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.category_name
```

**Answer**:

<img width="433" height="64" alt="image" src="https://github.com/user-attachments/assets/b59a77d8-e2e9-469e-a018-a495618d87be" />

**5. What is the top selling product for each category?**
```sql
WITH cte AS (
SELECT
  pd.category_name,
  pd.product_name,
  SUM (s.qty) as total_quantity,
  RANK () OVER (PARTITION BY pd.category_name ORDER BY SUM (s.qty) DESC) as ranking
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.category_name, pd.product_name
)
SELECT
  category_name,
  product_name,
  total_quantity
FROM cte
WHERE ranking = 1
```

**Answer**:

<img width="397" height="62" alt="image" src="https://github.com/user-attachments/assets/b7b3d4b0-7cae-4e85-b795-0e04ddbfdbce" />

**6. What is the percentage split of revenue by product for each segment?**
```sql
SELECT
  pd.segment_name,
  pd.product_name,
  ROUND (SUM (s.qty * s.price) / SUM (SUM (s.qty * s.price)) OVER (PARTITION BY pd.segment_name) *100,2) as percentage
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.segment_name, pd.product_name
```

**Answer**:

**7. What is the percentage split of revenue by segment for each category?**
```sql
SELECT
  pd.category_name,
  pd.segment_name,
  ROUND (SUM (s.qty * s.price) / SUM (SUM (s.qty * s.price)) OVER (PARTITION BY pd.category_name) *100,2) as percentage
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.category_name, pd.segment_name
```

**Answer**:

**8. What is the percentage split of total revenue by category?**
```sql
SELECT
  pd.category_name,
  ROUND (SUM (s.qty * s.price) / SUM (SUM (s.qty * s.price)) OVER () *100,2) as percentage
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.category_name
```

**Answer**:

**9. What is the total transaction “penetration” for each product? (hint: penetration = number of transactions where at least 1 quantity of a product was purchased divided by total number of transactions)**
```sql
SELECT
  pd.product_name,
  COUNT (DISTINCT txn_id) as transactions_amount,
  ROUND (COUNT (DISTINCT txn_id) /
    (SELECT COUNT (DISTINCT txn_id) as total_transactions
    FROM `weeksqlchallenge-474608.7balancedtree.sales`) *100 , 2) as penetration
FROM `weeksqlchallenge-474608.7balancedtree.sales` as s
JOIN `weeksqlchallenge-474608.7balancedtree.product_details` as pd
ON s.product_id = pd.product_id
GROUP BY pd.product_name
ORDER BY penetration DESC
```

**Answer**:

**10. What is the most common combination of at least 1 quantity of any 3 products in a 1 single transaction?**
```sql

```

**Answer**:

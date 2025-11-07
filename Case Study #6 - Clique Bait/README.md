# Case Study #6 - Clique Bait

---

## 🗂️ Table of Contents
- [Business Context](#business-context)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Case Study Questions and Solutions](#case-study-questions-and-solutions)

---

## Business Context

Clique Bait is not like your regular online seafood store - the founder and CEO Danny, was also a part of a digital data analytics team and wanted to expand his knowledge into the seafood industry!

In this case study - you as a Data Analyst are required to support Danny’s vision and analyse his dataset and come up with creative solutions to calculate funnel fallout rates for the Clique Bait online store.

---

## Entity Relationship Diagram

Below is the **Entity Relationship Diagram (ERD)** illustrating Clique Bait’s database structure. For this case study there is a total of 5 datasets

<p align="center">
<img width="1650" height="948" alt="image" src="https://github.com/user-attachments/assets/5ec853ce-9cc8-4450-a330-3d9e6f68e322" />
</p>
<p align="center"><em>Figure 1: Entity Relationship Diagram (ERD) of Clique Bait database</em></p>

---

## Case Study Questions and Solutions

The analysis in this case study was performed with BigQuery as the main querying tool.

### A. Digital Analysis
**1. How many users are there?**
```sql
SELECT
  COUNT (DISTINCT user_id) as number_users
FROM `weeksqlchallenge-474608.6cliquebait.users`
```

**Answer**: There are 500 users

<img width="130" height="58" alt="image" src="https://github.com/user-attachments/assets/ccc0de69-e48c-4795-ba7d-557ad28c03ce" />

**2. How many cookies does each user have on average?**
```sql
WITH cookie AS (
SELECT
  user_id,
  COUNT (cookie_id) as number_cookie
FROM `weeksqlchallenge-474608.6cliquebait.users`
GROUP BY user_id)

SELECT
  ROUND (AVG (number_cookie)) as avg_cookie
FROM cookie
```

**Answer**: Each user has approximately 4 cookies.

<img width="130" height="57" alt="image" src="https://github.com/user-attachments/assets/51639da9-9789-470b-9b08-6f8c1199cf03" />

**3. What is the unique number of visits by all users per month?**
```sql
SELECT
  EXTRACT (MONTH from event_time) as month,
  COUNT (DISTINCT visit_id) as unique_visit_number
FROM `weeksqlchallenge-474608.6cliquebait.events`
GROUP BY month
ORDER BY month 
```

**Answer**:

<img width="267" height="165" alt="image" src="https://github.com/user-attachments/assets/a21342bd-3f41-441f-8053-37368e006138" />

**4. What is the number of events for each event type?**
```sql
SELECT
  event_type,
  COUNT (event_type) as number_of_events
FROM `weeksqlchallenge-474608.6cliquebait.events`
GROUP BY event_type
ORDER BY event_type
```

**Answer**:

<img width="250" height="162" alt="image" src="https://github.com/user-attachments/assets/c129aad9-4c4d-4248-8e5c-a614bf5a029a" />

**5. What is the percentage of visits which have a purchase event?**
```sql
SELECT
  ROUND (SUM (CASE WHEN event_type = 3 THEN 1 ELSE 0 END) / COUNT (DISTINCT visit_id) * 100,2) AS percentage
FROM `weeksqlchallenge-474608.6cliquebait.events`
```

**Answer**: Approximately 49.86% of total website visits converted into purchases.

<img width="129" height="56" alt="image" src="https://github.com/user-attachments/assets/78d180a3-67c3-442a-841b-735ddfb103d3" />

**6. What is the percentage of visits which view the checkout page but do not have a purchase event?**
```sql
WITH cte AS (
SELECT
  visit_id,
  SUM (CASE WHEN page_id = 12 and event_type = 1 then 1 ELSE 0 END) AS view_checkout_page,
  SUM (CASE WHEN event_type = 3 then 1 ELSE 0 END) AS purchase
FROM `weeksqlchallenge-474608.6cliquebait.events`
GROUP BY visit_id)

SELECT
  SUM (view_checkout_page) as total_visit_checkout_page,
  SUM (purchase) as total_purchase,
  SUM (view_checkout_page) - SUM (purchase) as checkout_only,
  ROUND ((SUM (view_checkout_page) - SUM (purchase))/ SUM (view_checkout_page) * 100,2) as percentage
FROM cte
```

**Answer**: 15.5% of users who reached the checkout page did not complete their purchase.

<img width="503" height="55" alt="image" src="https://github.com/user-attachments/assets/ec448dbc-8842-4255-8054-5c26269f6654" />

**7. What are the top 3 pages by number of views?**
```sql
SELECT
  p.page_name,
  SUM (CASE WHEN e.event_type = 1 THEN 1 ELSE 0 END) as view_number
FROM `weeksqlchallenge-474608.6cliquebait.events` as e
JOIN `weeksqlchallenge-474608.6cliquebait.page_hierarchy` as p
ON e.page_id = p.page_id
GROUP BY p.page_name
ORDER BY view_number DESC
LIMIT 3
```

**Answer**:

<img width="328" height="84" alt="image" src="https://github.com/user-attachments/assets/f92dab77-86d8-44a3-8006-44809d763ad6" />

**8. What is the number of views and cart adds for each product category?**
```sql
SELECT
  p.product_category,
  SUM (CASE WHEN e.event_type = 1 THEN 1 ELSE 0 END) as view_number,
  SUM (CASE WHEN e.event_type = 2 THEN 1 ELSE 0 END) as card_add_number
FROM `weeksqlchallenge-474608.6cliquebait.events` as e
JOIN `weeksqlchallenge-474608.6cliquebait.page_hierarchy` as p
ON e.page_id = p.page_id
WHERE p.product_category is NOT NULL
GROUP BY p.product_category
```

**Answer**:

<img width="450" height="111" alt="image" src="https://github.com/user-attachments/assets/c0313733-804e-466f-bcf4-378815d18a27" />

**9. What are the top 3 products by purchases?**

- Firstly, I create cte1 to identifies all products added to cart in each visit, creating a mapping between visits and potential purchased products.
- Then, create cte2 is to lists all visits where a purchase occurred, enabling us to link those purchase visits with the added products from cte1.
- Purpose of both: I can determine which products were added to cart during visits that ended with a purchase — effectively identifying purchased products and counting them per product.

```sql
With cte1 AS (
SELECT
  e.visit_id,
  p.product_id
FROM `weeksqlchallenge-474608.6cliquebait.events` as e
JOIN `weeksqlchallenge-474608.6cliquebait.page_hierarchy` as p
ON e.page_id = p.page_id
WHERE event_type = 2
GROUP BY e.visit_id, p.product_id
),

cte2 AS (
SELECT 
  DISTINCT visit_id
FROM `weeksqlchallenge-474608.6cliquebait.events`
WHERE event_type = 3
) 
SELECT 
  product_id,
  COUNT (product_id) as purchase_number
FROM cte1 as c1
JOIN cte2 as c2
ON c1.visit_id = c2.visit_id
GROUP BY product_id
ORDER BY purchase_number DESC
LIMIT 3
```

**Answer**: Lobster, Oyster, and Crab are the three best-selling products

<img width="249" height="109" alt="image" src="https://github.com/user-attachments/assets/4755fe2b-2034-4101-b9fe-86a641a3b536" />

### B. Product Funnel Analysis

**1. Using a single SQL query - create a new output table which has the following details:**
- How many times was each product viewed?
- How many times was each product added to cart?
- How many times was each product added to a cart but not purchased (abandoned)?
- How many times was each product purchased?

**Answer**
- Firstly, joins events with page_hierarchy to map each visit_id to its product_id, using SUM (Case when) with condition like event_type = 1 (View) and event_type = 2 (Add to Cart)
- Then, aggregates visits with a purchase event (event_type = 3) 
- Join Step: JOIN the two CTEs on visit_id to link added/viewed products with visits that had a purchase.
- Final Aggregation: Use SUM() and conditional CASE WHEN logic to calculate per-product metrics: views, adds, abandoned carts (added=1 AND purchase=0), and purchases (added=1 AND purchase=1).

```sql
With view_and_cart_add AS (
SELECT
  e.visit_id,
  p.product_id,
  p.page_name,
  SUM (Case when event_type = 1 then 1 else 0 end) as view_page,
  SUM (Case when event_type = 2 then 1 else 0 end) as cart_add
FROM `weeksqlchallenge-474608.6cliquebait.events` as e
JOIN `weeksqlchallenge-474608.6cliquebait.page_hierarchy` as p
ON e.page_id = p.page_id
WHERE p.product_id is NOT NULL 
GROUP BY e.visit_id, p.product_id, p.page_name
),

purchase AS (
SELECT 
  visit_id,
  SUM (Case when event_type = 3 then 1 else 0 end) as purchase
FROM `weeksqlchallenge-474608.6cliquebait.events`
GROUP BY visit_id
), 

combine AS (
SELECT 
  a.visit_id,
  a.product_id,
  a.page_name,
  a.view_page,
  a.cart_add,
  b.purchase
FROM view_and_cart_add as a
JOIN purchase as b
ON a.visit_id = b.visit_id
)

SELECT 
  product_id,
  page_name,
  SUM (view_page) as view_number,
  SUM (cart_add) as cart_add_number,
  SUM (CASE WHEN cart_add = 1 and purchase = 0 then 1 else 0 end) as abandoned,
  SUM (CASE WHEN cart_add = 1 and purchase = 1 then 1 else 0 end) as purchase_number
FROM combine
GROUP BY product_id, page_name
ORDER BY product_id
```
<img width="824" height="272" alt="image" src="https://github.com/user-attachments/assets/1636ac4d-3ed2-44ae-9e5f-cc7c11159092" />


**2. Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.**

```sql
With view_and_cart_add AS (
SELECT
  e.visit_id,
  p.product_category,
  p.product_id,
  p.page_name,
  SUM (Case when event_type = 1 then 1 else 0 end) as view_page,
  SUM (Case when event_type = 2 then 1 else 0 end) as cart_add
FROM `weeksqlchallenge-474608.6cliquebait.events` as e
JOIN `weeksqlchallenge-474608.6cliquebait.page_hierarchy` as p
ON e.page_id = p.page_id
WHERE p.product_category is NOT NULL 
GROUP BY e.visit_id, p.product_category, p.product_id, p.page_name
),

purchase AS (
SELECT 
  visit_id,
  SUM (Case when event_type = 3 then 1 else 0 end) as purchase
FROM `weeksqlchallenge-474608.6cliquebait.events`
GROUP BY visit_id
), 

combine AS (
SELECT 
  a.visit_id,
  a.product_category,
  a.product_id,
  a.page_name,
  a.view_page,
  a.cart_add,
  b.purchase
FROM view_and_cart_add as a
JOIN purchase as b
ON a.visit_id = b.visit_id
)

SELECT 
  product_category,
  SUM (view_page) as view_number,
  SUM (cart_add) as cart_add_number,
  SUM (CASE WHEN cart_add = 1 and purchase = 0 then 1 else 0 end) as abandoned,
  SUM (CASE WHEN cart_add = 1 and purchase = 1 then 1 else 0 end) as purchase_number
FROM combine
GROUP BY product_category
ORDER BY product_category
```
<img width="701" height="109" alt="image" src="https://github.com/user-attachments/assets/12ba733f-78a8-4183-ae9a-c5296f262684" />

**3. Use your 2 new output tables - answer the following questions:**

**a. Which product had the most views, cart adds and purchases?**
- Oyster is the most viewed product with 1,568 page views, while Lobster has the highest number of add-to-cart actions (968) and purchases (754).

**b. Which product was most likely to be abandoned?**
- Russian Caviar was most likely to be abandoned

**c. Which product had the highest view to purchase percentage?**
```sql
SELECT
  product_id,
  page_name,
  view_number,
  purchase_number,
  ROUND (purchase_number/ view_number * 100, 2) as percentage
FROM product_funnel
ORDER BY percentage DESC
```
**Answer:** It's Lobster

<img width="697" height="267" alt="image" src="https://github.com/user-attachments/assets/cb3488f1-c715-4f94-bde2-9f3142a2e399" />

**d. What is the average conversion rate from view to cart add? And average conversion rate from cart add to purchase?**
```sql
SELECT
  ROUND (AVG (cart_add_number/ view_number * 100), 2) as avg_view_to_cart_add_conversion,
  ROUND (AVG (purchase_number/ cart_add_number * 100), 2) as avg_cart_add_to_purchase_conversion
FROM product_funnel
```
**Answer:**

<img width="461" height="56" alt="image" src="https://github.com/user-attachments/assets/5f0d1af1-0069-487c-bf8b-6a1bcf3bffea" />

### C.  Campaigns Analysis





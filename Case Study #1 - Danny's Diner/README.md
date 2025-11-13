# Case Study #1 - Danny's Diner

---

## 🗂️ Table of Contents
- [Business Context](#business-context)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Case Study Questions and Solutions](#case-study-questions-and-solutions)

---

## Business Context

Danny opened Danny’s Diner Restaurant in early 2021, serving sushi, curry, and ramen, but the team lacks the skills to analyze their early customer data for better business performance.

Danny wants to understand his customers better — their visit patterns, spending habits, and favorite dishes. These insights will help him personalize the customer experience and decide whether to expand the loyalty program.

Your task is to use Danny’s sample data to create datasets, write SQL queries, and generate actionable insights for business decisions.

---

## Entity Relationship Diagram

Below is the **Entity Relationship Diagram (ERD)** illustrating Danny's Diner’s database structure. For this case study there are 3 key datasets: sales, members, menu

<p align="center">
  <img width="526" height="260" alt="image" src="https://github.com/user-attachments/assets/95f590ea-3ff3-4999-a9cf-e9661eb63a2c" />
</p>

<p align="center"><em>Figure 1: Entity Relationship Diagram (ERD) of Danny's Diner database</em></p>

---

## Case Study Questions and Solutions

The analysis in this case study was performed with BigQuery as the main querying tool.

### A. Case Study Questions

**1. What is the total amount each customer spent at the restaurant?**

```sql
SELECT
  s.customer_id,
  SUM (m.price) as total_spend
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as m
ON s.product_id = m.product_id
GROUP BY s.customer_id
```

**Answer**: Customers A, B, and C spent 76 USD, 74 USD, and 36 USD respectively at Danny’s Diner.

<img width="167" height="82" alt="image" src="https://github.com/user-attachments/assets/dfbea11a-2d59-4cc6-befc-fafe31db6c82" />


**2. How many days has each customer visited the restaurant?**

```sql
SELECT
  customer_id,
  COUNT (DISTINCT order_date) as days
FROM `weeksqlchallenge-474608.1dannysdiner.sales`
GROUP BY customer_id
ORDER BY days DESC
```

**Answer**: Customers B, A, and C visited Danny’s Diner 6, 4, and 2 times respectively.

<img width="150" height="81" alt="image" src="https://github.com/user-attachments/assets/f575a500-278b-4ead-9e98-02f4e31f5759" />


**3. What was the first item from the menu purchased by each customer?**

- First, create a CTE to rank the items each customer ordered chronologically, from earliest to latest.
- Use DENSE_RANK() so that tied values receive the same rank without skipping the next rank (e.g., 1, 2, 2, 3).
- Make sure to combine DENSE_RANK() with PARTITION BY customer_id and ORDER BY order_date ASC to rank orders separately for each customer in ascending order of date.

```sql
With order_ranking AS (
SELECT
  s.customer_id,
  m.product_name,
  s.order_date,
  DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY s.order_date ASC) AS order_rank
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as m
ON s.product_id = m.product_id
GROUP BY s.customer_id,  m.product_name, s.order_date)

SELECT
  customer_id,
  order_date,
  product_name
FROM order_ranking
WHERE order_rank = 1
```

**Answer**: Customer A’s first order included both curry and sushi, while Customer B started with curry and Customer C with ramen.

<img width="288" height="104" alt="image" src="https://github.com/user-attachments/assets/d1d59212-05ff-4088-ab2d-2601a0333ff3" />

**4. What is the most purchased item on the menu and how many times was it purchased by all customers?**

```sql
SELECT
  m.product_name,
  COUNT (m.product_name) as purchase_number
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as m
ON s.product_id = m.product_id
GROUP BY m.product_name
ORDER BY purchase_number DESC
```

**Answer**: Ramen is the most purchased item at the restaurant, with a total of 8 orders.

<img width="247" height="85" alt="image" src="https://github.com/user-attachments/assets/bd03c3c4-9a02-40d6-9b30-53f27f87b9c6" />


**5. Which item was the most popular for each customer?**

```sql
With ranking AS (
SELECT
  s.customer_id,
  m.product_name,
  COUNT (m.product_name) as purchase_number,
  DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY COUNT (m.product_name) DESC) as rank
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as m
ON s.product_id = m.product_id
GROUP BY s.customer_id, m.product_name)

SELECT 
  customer_id,
  product_name,
  purchase_number
FROM ranking
WHERE rank = 1
ORDER BY customer_id
```

**Answer**: Customers A and C order ramen most frequently, while Customer B orders all three items equally.

<img width="246" height="114" alt="image" src="https://github.com/user-attachments/assets/7d9a277a-a341-4c84-8565-beda33da22e5" />

**6. Which item was purchased first by the customer after they became a member?**

```sql
---Which item was purchased first by the customer after they became a member?
With cte AS (
SELECT
  s.customer_id,
  s.order_date,
  mn.product_name,
  DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY s.order_date ASC) AS ranking
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as mn
ON s.product_id = mn.product_id
JOIN `weeksqlchallenge-474608.1dannysdiner.members` as mb
ON s.customer_id = mb.customer_id
WHERE s.order_date >= mb.join_date
GROUP BY s.customer_id, s.order_date, mn.product_name
)

SELECT 
 *
FROM cte
WHERE ranking = 1
```

**Answer**: Customer A first ordered curry as a member, whereas Customer B’s first member order was sushi.

<img width="315" height="61" alt="image" src="https://github.com/user-attachments/assets/eecfd3d6-4a54-4faa-b145-ae8fadc15824" />


**7. Which item was purchased just before the customer became a member?**

```sql
With cte AS (
SELECT
  s.customer_id,
  s.order_date,
  mn.product_name,
  DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY s.order_date DESC) AS ranking
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as mn
ON s.product_id = mn.product_id
JOIN `weeksqlchallenge-474608.1dannysdiner.members` as mb
ON s.customer_id = mb.customer_id
WHERE s.order_date < mb.join_date
GROUP BY s.customer_id, s.order_date, mn.product_name
)

SELECT 
 *
FROM cte
WHERE ranking = 1
```

**Answer**: For Customer A, the item purchased just before becoming a member were sushi and curry, and for Customer B, it was sushi only.

<img width="332" height="77" alt="image" src="https://github.com/user-attachments/assets/e28c3186-b366-4201-b3af-58f3c924a029" />

**8. What is the total items and amount spent for each member before they became a member?**

```sql
SELECT
  s.customer_id,
  COUNT (s.product_id) as total_items,
  SUM (mn.price) as total_spend
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as mn
ON s.product_id = mn.product_id
JOIN `weeksqlchallenge-474608.1dannysdiner.members` as mb
ON s.customer_id = mb.customer_id
WHERE s.order_date < mb.join_date
GROUP BY s.customer_id
```

**Answer**: Before becoming a member, Customer A purchased 2 items totaling 25 USD, and Customer B purchased 3 items totaling 40 USD.

<img width="256" height="60" alt="image" src="https://github.com/user-attachments/assets/ab4abf1b-a7da-45e8-a2dd-60b80db771a1" />

**9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?**

- Create a new column called points_multiplier using a CASE WHEN statement: assign a value of 10 if the product is curry or ramen, and 20 if the product is sushi.
- Then, calculate the total points for each customer using `SUM (price * points_multiplier)` and `group by customer_id`.

```sql
With cte AS (
SELECT
  s.customer_id,
  s.product_id,
  m.price,
  (CASE
    WHEN s.product_id = 1 THEN 20
    WHEN s.product_id = 2 THEN 10
    WHEN s.product_id = 3 THEN 10
    ELSE 0
  END) AS points_multiplier
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as m
ON s.product_id = m.product_id)

SELECT
  customer_id,
  SUM (price * points_multiplier) as total_point
FROM cte
GROUP BY customer_id
```

**Answer**: Based on the points system where each $1 equals 10 points and sushi earns 20 points, Customer A has 860 points, Customer B has 940 points, and Customer C has 360 points.

<img width="205" height="84" alt="image" src="https://github.com/user-attachments/assets/66414784-3961-4a2d-9a92-c1c0787d1dc2" />


**10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?**

- Create the points_multiplier column using a CASE WHEN statement with two conditions:

*Condition 1:* During the first week after a customer joins (including the join date), they earn 2x points on all items.

→ Use: `order_date BETWEEN join_date AND DATE_ADD(join_date, INTERVAL 7 DAY)`

*Condition 2:* For all other dates, the points multiplier remains the same.

→ Use: `order_date NOT BETWEEN join_date AND DATE_ADD(join_date, INTERVAL 7 DAY)`

- Next, filter all records with order_date up to the end of January

→ Use: `WHERE order_date < '2021-01-31'`

- The remaining steps should be performed the same way as in Question 9.

```sql
With cte AS (
SELECT
  s.customer_id,
  s.order_date,
  mb.join_date,
  s.product_id,
  mn.price,
  (CASE
    WHEN s.order_date BETWEEN mb.join_date AND DATE_ADD(mb.join_date, INTERVAL 7 DAY) THEN 20
    WHEN s.order_date NOT BETWEEN mb.join_date AND DATE_ADD(mb.join_date, INTERVAL 7 DAY) and s.product_id = 1 THEN 20
    WHEN s.order_date NOT BETWEEN mb.join_date AND DATE_ADD(mb.join_date, INTERVAL 7 DAY) and s.product_id = 2 THEN 10
    WHEN s.order_date NOT BETWEEN mb.join_date AND DATE_ADD(mb.join_date, INTERVAL 7 DAY) and s.product_id = 3 THEN 10
    ELSE 0
  END) AS points_multiplier
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as mn
ON s.product_id = mn.product_id
JOIN `weeksqlchallenge-474608.1dannysdiner.members` as mb
ON s.customer_id = mb.customer_id
WHERE s.order_date <= '2021-01-31')

SELECT
  customer_id,
  SUM (price * points_multiplier) as total_point
FROM cte
GROUP BY customer_id
```

**Answer**:

<img width="192" height="63" alt="image" src="https://github.com/user-attachments/assets/a1cb632b-4d20-4b09-85eb-0f207e956415" />

### B. Bonus Questions

**1. Join all the things**

```sql
SELECT
  s.customer_id,
  s.order_date,
  mn.product_name,
  mn.price,
  (CASE WHEN s.order_date >= mb.join_date THEN 'Y' ELSE 'N' END) AS member
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as mn
ON s.product_id = mn.product_id
LEFT JOIN `weeksqlchallenge-474608.1dannysdiner.members` as mb
ON s.customer_id = mb.customer_id
```
**Answer**: 

| customer_id | order_date  | product_name | price | member |
|--------------|-------------|---------------|--------|---------|
| A | 2021-01-01 | curry | 15 | N |
| A | 2021-01-01 | sushi | 10 | N |
| A | 2021-01-07 | curry | 15 | Y |
| A | 2021-01-10 | ramen | 12 | Y |
| A | 2021-01-11 | ramen | 12 | Y |
| A | 2021-01-11 | ramen | 12 | Y |
| B | 2021-01-01 | curry | 15 | N |
| B | 2021-01-02 | curry | 15 | N |
| B | 2021-01-04 | sushi | 10 | N |
| B | 2021-01-11 | sushi | 10 | Y |
| B | 2021-01-16 | ramen | 12 | Y |
| B | 2021-02-01 | ramen | 12 | Y |
| C | 2021-01-01 | ramen | 12 | N |
| C | 2021-01-01 | ramen | 12 | N |
| C | 2021-01-07 | ramen | 12 | N |

**2. Rank all the things**
```sql
WITH customer_data AS 
(SELECT
  s.customer_id,
  s.order_date,
  mn.product_name,
  mn.price,
  (CASE
    WHEN s.order_date >= mb.join_date THEN 'Y'
    ELSE 'N' END) AS member
FROM `weeksqlchallenge-474608.1dannysdiner.sales` as s
JOIN `weeksqlchallenge-474608.1dannysdiner.menu` as mn
ON s.product_id = mn.product_id
LEFT JOIN `weeksqlchallenge-474608.1dannysdiner.members` as mb
ON s.customer_id = mb.customer_id)

SELECT
  *,
  (CASE WHEN member = 'Y' THEN DENSE_RANK() OVER (PARTITION BY customer_id, member ORDER BY order_date)
  ELSE null END) AS ranking
FROM customer_data
```

**Answer**: 

| customer_id | order_date  | product_name | price | member | ranking |
|--------------|-------------|---------------|--------|---------|----------|
| A | 2021-01-01 | curry | 15 | N | null |
| A | 2021-01-01 | sushi | 10 | N | null |
| A | 2021-01-07 | curry | 15 | Y | 1 |
| A | 2021-01-10 | ramen | 12 | Y | 2 |
| A | 2021-01-11 | ramen | 12 | Y | 3 |
| A | 2021-01-11 | ramen | 12 | Y | 3 |
| B | 2021-01-01 | curry | 15 | N | null |
| B | 2021-01-02 | curry | 15 | N | null |
| B | 2021-01-04 | sushi | 10 | N | null |
| B | 2021-01-11 | sushi | 10 | Y | 1 |
| B | 2021-01-16 | ramen | 12 | Y | 2 |
| B | 2021-02-01 | ramen | 12 | Y | 3 |
| C | 2021-01-01 | ramen | 12 | N | null |
| C | 2021-01-01 | ramen | 12 | N | null |
| C | 2021-01-07 | ramen | 12 | N | null |


# Case Study #3 - Foodie-Fi

---

## 🗂️ Table of Contents
- [Business Context](#business-context)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Case Study Questions and Solutions](#case-study-questions-and-solutions)

---

## Business Context

Subscription businesses are very popular, and Danny saw a big opportunity in creating a food-only streaming service. His idea was a Netflix-style platform that features only cooking shows. 

In 2020, Danny and a few friends launched a startup called Foodie-Fi and began selling monthly and annual plans. Customers get unlimited access to exclusive food-related videos.

Danny wanted the company to be data-driven from day one to support smart decision-making. This case study explores how subscription data can be used to answer key business questions.

---

## Entity Relationship Diagram

Below is the **Entity Relationship Diagram (ERD)** illustrating Foodie-Fi’s database structure. For this case study there are 2 tables: `plans` and `subscriptions`

<p align="center">
  <img width="1018" height="415" alt="image" src="https://github.com/user-attachments/assets/19de7191-3f13-42dc-bcdf-1bd5e0d6604e" />
</p>

<p align="center"><em>Figure 1: Entity Relationship Diagram (ERD) of Foodie-Fi database</em></p>


---

## Case Study Questions and Solutions

The analysis in this case study was performed with BigQuery as the main querying tool.

### A. Customer Journey

### B. Data Analysis Questions

**1. How many customers has Foodie-Fi ever had?**
```sql
SELECT
  COUNT (DISTINCT customer_id) as total_customers
FROM `weeksqlchallenge-474608.3foodiefi.subscriptions`
```

**Answer**: Foodie-Fi has 1000 customers.

<img width="140" height="62" alt="image" src="https://github.com/user-attachments/assets/578d4e6b-61f2-4be6-a1b0-cd1d157dc1f3" />

**2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value**
```sql
SELECT
  EXTRACT (MONTH from start_date) as month,
  COUNT (customer_id) as total_customers
FROM `weeksqlchallenge-474608.3foodiefi.subscriptions`
WHERE plan_id = 0
GROUP BY month
ORDER BY month
```

**Answer**:

<img width="234" height="387" alt="image" src="https://github.com/user-attachments/assets/a77a4bd8-595a-439a-aadc-b7717481a8c6" />

**3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name**
```sql
SELECT
  p.plan_name,
  COUNT (customer_id) as total_customers
FROM `weeksqlchallenge-474608.3foodiefi.subscriptions` as s
JOIN `weeksqlchallenge-474608.3foodiefi.plans` as p
ON s.plan_id = p.plan_id
WHERE start_date >= '2021-01-01'
GROUP BY plan_name
ORDER BY total_customers DESC
```

**Answer**: After 2020, the largest customer segment was those who churned, with a total of 71 users. The smallest group consisted of customers who subscribed to the Basic Monthly plan, with only 8 users. Notably, no customers signed up for the Trial plan. Despite this, the Pro Monthly and Pro Annual plans were highly popular, attracting a combined total of 123 users.

<img width="279" height="153" alt="image" src="https://github.com/user-attachments/assets/1e914576-1480-4b85-a1a6-26555c19be54" />

**4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?**

**Definition of churn customers**: A churn customer is someone who, after canceling their subscription, does not purchase any additional plans on Foodie-Fi. Their final recorded plan must be Churn. If a customer cancels but later signs up for any new subscription, they should not be classified as a churn customer.

Therefore, I will approach this question using the following three steps:
- Use `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date DESC)` to rank each customer’s subscriptions from the most recent to the oldest.
- Forming the “last plan” table by using filter for `WHERE ranking = 1` to extract each customer’s most recent subscription.
- Then compute the number and percentage of churn customers whose last plan is `plan_id = 4`.

```sql
With ranking AS (
  SELECT
    customer_id,
    plan_id,
    start_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date DESC) as rank
  FROM `weeksqlchallenge-474608.3foodiefi.subscriptions`
),
 
last_plan AS (
  SELECT
    customer_id,
    plan_id,
    start_date
  FROM ranking
  WHERE rank = 1
)

SELECT
  COUNT (DISTINCT customer_id) as total_customers,
  COUNT(DISTINCT CASE WHEN plan_id = 4 THEN customer_id END) AS churn_customers,
  ROUND(COUNT(DISTINCT CASE WHEN plan_id = 4 THEN customer_id END)/ COUNT(DISTINCT customer_id) * 100,1) AS churn_percentage
FROM last_plan
```

**Answer**: 307 out of 1,000 customers churned, which is 30.7%.

<img width="414" height="61" alt="image" src="https://github.com/user-attachments/assets/e6bb6c57-f67d-46a7-b988-cb1b56946967" />

**5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?**
```sql
With ranking AS (
  SELECT
    customer_id,
    plan_id,
    start_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date) as rank
  FROM `weeksqlchallenge-474608.3foodiefi.subscriptions`
),
 
plans AS (
  SELECT
    customer_id,
    MAX(CASE WHEN rank = 1 then plan_id END) AS first_plan,
    MAX(CASE WHEN rank = 2 then plan_id END) AS second_plan
  FROM ranking
  GROUP BY customer_id
  ORDER BY customer_id
)

SELECT 
  COUNT (CASE WHEN first_plan = 0 and second_plan = 4 THEN customer_id END) AS total_churn_after_trial,
  ROUND ((COUNT (CASE WHEN first_plan = 0 and second_plan = 4 THEN customer_id END) / COUNT (*) * 100),0) AS percentage
FROM plans
```

**Answer**: 92 customers churned immediately after their initial free trial, representing 9% of the total.

<img width="309" height="60" alt="image" src="https://github.com/user-attachments/assets/b2d6b737-4031-4176-ba8c-b515b9048f19" />

**6. What is the number and percentage of customer plans after their initial free trial?**
```sql
With ranking AS (
  SELECT
    customer_id,
    plan_id,
    start_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date) as rank
  FROM `weeksqlchallenge-474608.3foodiefi.subscriptions`
),
 
plans AS (
  SELECT
    customer_id,
    MAX(CASE WHEN rank = 1 then plan_id END) AS first_plan,
    MAX(CASE WHEN rank = 2 then plan_id END) AS second_plan
  FROM ranking
  GROUP BY customer_id
  ORDER BY customer_id
)

SELECT 
  plan_name,
  COUNT (customer_id) as total,
  ROUND (COUNT (customer_id) / SUM (COUNT (customer_id)) OVER () *100,2) as percentage
FROM plans as p1
JOIN `weeksqlchallenge-474608.3foodiefi.plans` as p2
ON p1.second_plan = p2.plan_id
GROUP BY plan_name
ORDER BY percentage DESC
```

**Answer**: More than 80% of Foodie-Fi customers upgrade to either the Basic Monthly or Pro Monthly plan right after the trial. About 9.2% churn, and approximately 3.7% choose the Pro Annual plan.

<img width="348" height="153" alt="image" src="https://github.com/user-attachments/assets/702caa49-a662-4585-80b8-bde55f9edc8b" />

**7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?**
```sql
With ranking AS (
  SELECT
    customer_id,
    plan_id,
    start_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date DESC) as rank
  FROM `weeksqlchallenge-474608.3foodiefi.subscriptions`
  WHERE start_date <= '2020-12-31'
)

SELECT 
  r.plan_id,
  plan_name,
  COUNT (customer_id) as customer_count,
  ROUND (COUNT (customer_id) / SUM (COUNT (customer_id)) OVER () *100,2) as percentage
FROM ranking as r
JOIN `weeksqlchallenge-474608.3foodiefi.plans` as p
ON r.plan_id = p.plan_id
WHERE rank = 1
GROUP BY plan_id, plan_name
ORDER BY plan_id
```

**Answer**:

<img width="457" height="180" alt="image" src="https://github.com/user-attachments/assets/38dfd18a-8df0-4343-9484-ada68f228a28" />

**8. How many customers have upgraded to an annual plan in 2020?**
```sql
SELECT
  COUNT (DISTINCT customer_id) as customer_count
FROM `weeksqlchallenge-474608.3foodiefi.subscriptions` 
WHERE start_date < '2021-01-01' and plan_id = 3
```

**Answer**: 195 customers have upgraded to an annual plan in 2020.

<img width="140" height="61" alt="image" src="https://github.com/user-attachments/assets/10215325-4376-4b75-a30d-d1911612a85a" />

**9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?**
```sql
With ranking AS (
  SELECT
    customer_id,
    plan_id,
    start_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date) as rank
  FROM `weeksqlchallenge-474608.3foodiefi.subscriptions`
),

annual_plan AS (
SELECT 
  customer_id,
  MAX (CASE WHEN rank = 1 THEN start_date END) as join_date,
  MAX (CASE WHEN plan_id = 3 THEN start_date END) as annual_plan_date
FROM ranking
GROUP BY customer_id
ORDER BY customer_id
)

SELECT
  ROUND (AVG (DATETIME_DIFF(annual_plan_date, join_date, DAY))) as day_difference
FROM annual_plan
WHERE annual_plan_date IS NOT NULL
```

**Answer**: It takes about 105 days on average for a customer to subscribe to an annual plan from the day they join Foodie-Fi.

<img width="143" height="59" alt="image" src="https://github.com/user-attachments/assets/8c45b828-82bb-4e6f-9c64-082c2530fe2a" />

**10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)**
```sql
With ranking AS (
  SELECT
    customer_id,
    plan_id,
    start_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date) as rank
  FROM `weeksqlchallenge-474608.3foodiefi.subscriptions`
),

annual_plan AS (
SELECT 
  customer_id,
  MAX (CASE WHEN rank = 1 THEN start_date END) as join_date,
  MAX (CASE WHEN plan_id = 3 THEN start_date END) as annual_plan_date
FROM ranking
GROUP BY customer_id
ORDER BY customer_id
),
days_to_annual_plan AS (
SELECT
  customer_id,
  DATETIME_DIFF(annual_plan_date, join_date, DAY) as days_to_annual
FROM annual_plan
WHERE annual_plan_date IS NOT NULL
)

SELECT
  (CASE 
    WHEN days_to_annual BETWEEN 0 AND 30 THEN '0-30 days'
    WHEN days_to_annual BETWEEN 31 AND 60 THEN '31-60 days'
    WHEN days_to_annual BETWEEN 61 AND 90 THEN '61-90 days'
    WHEN days_to_annual BETWEEN 91 AND 120 THEN '91-120 days'
    WHEN days_to_annual BETWEEN 121 AND 150 THEN '121-150 days'
    ELSE 'Over 150 days' END) AS periods,
  COUNT (customer_id) as total_customer
FROM days_to_annual_plan
GROUP BY periods
ORDER BY periods
```

**Answer**:

<img width="288" height="212" alt="image" src="https://github.com/user-attachments/assets/c47fa700-9bee-4774-8a65-64f0b3c946ef" />

**11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?**

To solve this question, I used `LEAD (colume_name) OVER (PARTITION BY ORDER BY)`
- The LEAD() function in BigQuery is a window function used to access data from the next row within the same result set, based on a specified ordering.
- This function is very useful for tasks such as tracking upgrades or downgrades (like in the Foodie-Fi case), calculating the time between purchases, and finding the amount of the next transaction.

```sql
WITH plans AS (
  SELECT
    customer_id,
    plan_id,
    start_date,
    LEAD (plan_id) OVER (PARTITION BY customer_id ORDER BY start_date) as next_plan_id,
    LEAD (start_date) OVER (PARTITION BY customer_id ORDER BY start_date) as next_plan_start_date
  FROM `weeksqlchallenge-474608.3foodiefi.subscriptions`
),
downgraded AS (
SELECT *
FROM plans
WHERE plan_id = 2 and next_plan_id = 1
)

SELECT
  COUNT (DISTINCT customer_id) as total_downgrade
FROM downgraded
```

**Answer**: In 2020, no customers switched from a Pro Monthly plan down to a Basic Monthly plan.

<img width="140" height="63" alt="image" src="https://github.com/user-attachments/assets/12e04ad8-86ec-4115-bf19-2040ac833602" />

### C. Challenge Payment Question

- Attach pricing and plan names, and generate the next_start_date column using
`LEAD() OVER (PARTITION BY customer_id ORDER BY start_date)`.

- Define each plan’s effective period using `valid_from = start_date` and
valid_to derived with `COALESCE(next_start_date, end_of_data_range)`.

- Restrict the active period to the 2020 calendar year by applying `GREATEST` and `LEAST` to clamp the date boundaries.

- Generate monthly payments for basic monthly and pro monthly plans using
`UNNEST(GENERATE_DATE_ARRAY(period_start, period_end, INTERVAL 1 MONTH))`.

- Generate annual payments for pro annual plans by charging once at the start_date.

- Combine all payments (excluding churn records since price is NULL and therefore filtered out upstream).

- Produce the final output as the payments_2020 table.

```sql
CREATE TABLE `weeksqlchallenge-474608.3foodiefi.payments_2020` AS

WITH sub_with_price AS (
  SELECT
    s.customer_id,
    s.plan_id,
    p.plan_name,
    p.price,
    s.start_date,
    LEAD(s.start_date) OVER (PARTITION BY s.customer_id ORDER BY s.start_date) AS next_start_date
  FROM `weeksqlchallenge-474608.3foodiefi.subscriptions` AS s
  JOIN `weeksqlchallenge-474608.3foodiefi.plans` AS p
  On s.plan_id = p.plan_id
),

plan_periods AS (
  SELECT
    customer_id,
    plan_id,
    plan_name,
    price,
    start_date AS valid_from,
    COALESCE(
      DATE_SUB(next_start_date, INTERVAL 1 DAY),
      DATE '2020-12-31') AS valid_to
  FROM sub_with_price
),

periods_2020 AS (
  SELECT
    customer_id,
    plan_id,
    plan_name,
    price,
    GREATEST(valid_from, DATE '2020-01-01') AS period_start,
    LEAST(valid_to, DATE '2020-12-31') AS period_end
  FROM plan_periods
  WHERE valid_from <= DATE '2020-12-31' AND valid_to >= DATE '2020-01-01'
),

monthly_payments AS (
  SELECT
    customer_id,
    plan_id,
    plan_name,
    price AS amount,
    payment_date
  FROM periods_2020,
  UNNEST(GENERATE_DATE_ARRAY(period_start,period_end, INTERVAL 1 MONTH)) AS payment_date
  WHERE plan_name IN ('basic monthly', 'pro monthly')
),

annual_payments AS (
  SELECT
    customer_id,
    plan_id,
    plan_name,
    price AS amount,
    period_start AS payment_date
  FROM periods_2020
  WHERE plan_name = 'pro annual'
),

all_payments AS (
  SELECT * FROM monthly_payments
  UNION ALL
  SELECT * FROM annual_payments
)

SELECT
  customer_id,
  plan_id,
  plan_name,
  payment_date,
  amount
FROM all_payments
ORDER BY customer_id, payment_date;
```
**Answer**:

<img width="664" height="327" alt="image" src="https://github.com/user-attachments/assets/d3a54552-0fe5-477f-a349-f0b57841fbc7" />


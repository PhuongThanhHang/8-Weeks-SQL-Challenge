# Case Study #1 - Pizza Runner

---

## 🗂️ Table of Contents
- [Business Context](#business-context)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Case Study Questions and Solutions](#case-study-questions-and-solutions)

---

## Business Context

Danny discovered that pizza is one of the most consumed foods in the world (over 115 million kilograms of pizza is consumed daily worldwide). Inspired by a social media post, he came up with the idea of creating a retro-style pizza brand. He launched Pizza Runner, combining pizza delivery with an Uber-like model. To start, he recruited runners for delivery and hired developers to build a mobile ordering app to accept orders from customers.

---

## Entity Relationship Diagram

Below is the **Entity Relationship Diagram (ERD)** illustrating Pizza Runner’s database structure. For this case study there are 6 key datasets: runners, customer_orders, runner_orders, pizza_names, pizza_recipes, pizza_toppings

<p align="center">
<img width="1056" height="521" alt="image" src="https://github.com/user-attachments/assets/198a36cb-c6d6-4edc-a09c-3befbf619e74" />
</p>

<p align="center"><em>Figure 1: Entity Relationship Diagram (ERD) of Pizza Runner database</em></p>

---

## Case Study Questions and Solutions

The analysis in this case study was performed with BigQuery as the main querying tool.

### 0. Data Cleaning

**1. Creating a new table from table customer_orders and cleaning table by replacing missing value and NAN value with 0**

```sql
CREATE TABLE `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` AS 
  SELECT 
  order_id, 
  customer_id, 
  pizza_id, 
  (CASE WHEN exclusions IS NULL OR exclusions LIKE 'null' OR exclusions = '' THEN '0'
    ELSE exclusions END) AS exclusions,
  (CASE WHEN extras IS NULL OR extras LIKE 'null' OR extras = '' THEN '0'
    ELSE extras END) AS extras,
  order_time
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders`
```

**2. Creating a new table from table runner_orders and cleaning table by replacing missing value and NAN value with 0**

```sql
CREATE TABLE `weeksqlchallenge-474608.2pizzarunner.runner_orders_clean` AS
SELECT
  order_id,
  runner_id,
  (CASE WHEN pickup_time IN ('null', 'NULL', '') THEN TIMESTAMP('0000-01-01 00:00:00')
    ELSE TIMESTAMP(pickup_time) END) AS pickup_time,
  CAST(CASE WHEN distance LIKE '%km%' THEN REPLACE(REPLACE(distance, 'km', ''), ' ', '')
            WHEN distance IN ('null', 'NULL', '') OR distance IS NULL THEN '0'
      ELSE distance END AS FLOAT64) AS distance,
  CAST(CASE WHEN duration LIKE '%minute%' THEN REPLACE(REPLACE(duration, 'minute', ''), 's', '')         WHEN duration LIKE '%mins%' THEN REPLACE(duration, 'mins', '')
            WHEN duration IN ('null', 'NULL', '') OR duration IS NULL THEN '0'
      ELSE duration END AS INT64) AS duration,
  (CASE WHEN cancellation IS NULL OR cancellation IN ('', 'null', 'NULL', 'NAN') THEN 'Successful'
    ELSE cancellation END) AS cancellation
FROM `weeksqlchallenge-474608.2pizzarunner.runner_orders`
```

### A. Case Study Questions

**1. How many pizzas were ordered?**
```sql
SELECT
  COUNT (pizza_id) as total_pizza
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new`
```

**Answer**: There were 14 pizzas orderd

<img width="139" height="60" alt="image" src="https://github.com/user-attachments/assets/6a6c4557-4350-4753-b269-dc8f2be91612" />

**2. How many unique customer orders were made?**
```sql
SELECT
  COUNT (DISTINCT order_id) as unique_orders
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new`
```

**Answer**: There were 10 unique orders made

<img width="134" height="60" alt="image" src="https://github.com/user-attachments/assets/fa2aa1e5-d38b-4bfc-8d6d-ee4638e813bc" />

**3. How many successful orders were delivered by each runner?**
```sql
SELECT
  runner_id,
  COUNT (DISTINCT order_id) as successful_orders
FROM `weeksqlchallenge-474608.2pizzarunner.runner_orders_new`
WHERE cancellation = 'Successful'
GROUP BY runner_id
ORDER BY runner_id
```

**Answer**:

<img width="276" height="123" alt="image" src="https://github.com/user-attachments/assets/2eac47c3-6b68-4140-a38c-29a24a3095bf" />

**4. How many of each type of pizza was delivered?**
```sql
SELECT
  p.pizza_name,
  COUNT (c.pizza_id) AS successfull_delivered
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as c
JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as r
ON c.order_id = r.order_id
JOIN `weeksqlchallenge-474608.2pizzarunner.pizza_names` as p
ON c.pizza_id = p.pizza_id
WHERE cancellation = 'Successful'
GROUP BY p.pizza_name
ORDER BY p.pizza_name
```

**Answer**: There were 9 Meatlovers pizzas and 3 Vegetarian pizzas delivered successfully

<img width="279" height="90" alt="image" src="https://github.com/user-attachments/assets/54423070-6617-4c8f-947e-d1fdc5fa6b10" />

**5. How many Vegetarian and Meatlovers were ordered by each customer?**
```sql
SELECT
  customer_id,
  SUM (CASE WHEN pizza_id = 1 then 1 ELSE 0 END) as Meat_Lovers,
  SUM (CASE WHEN pizza_id = 2 then 1 ELSE 0 END) as Vegetarian
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` 
GROUP BY customer_id
ORDER BY customer_id
```

**Answer**:

<img width="411" height="181" alt="image" src="https://github.com/user-attachments/assets/33c772f2-a134-4d5d-be8e-80a405c5cc59" />

**6. What was the maximum number of pizzas delivered in a single order?**
```sql
SELECT
  c.order_id,
  COUNT (c.pizza_id) AS successfull_delivered
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as c
JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as r
ON c.order_id = r.order_id
WHERE cancellation = 'Successful'
GROUP BY c.order_id
ORDER BY successfull_delivered DESC
```

**Answer**: The maximum number of pizzas delivered in a single order is 3

<img width="303" height="267" alt="image" src="https://github.com/user-attachments/assets/f757f3a6-53d7-4656-b14e-723f5039d716" />

**7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?**
```sql
SELECT
  c.customer_id,
  SUM (CASE WHEN exclusions != '0' OR extras != '0' THEN 1 ELSE 0 END) as change,
  SUM (CASE WHEN exclusions = '0' AND extras = '0' THEN 1 ELSE 0 END) as no_change
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as c
JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as r
ON c.order_id = r.order_id
WHERE cancellation = 'Successful'
GROUP BY c.customer_id
ORDER BY c.customer_id
```

**Answer**:

<img width="309" height="183" alt="image" src="https://github.com/user-attachments/assets/95f51922-a821-4325-962d-16b6dc8dbd59" />

**8. How many pizzas were delivered that had both exclusions and extras?**
```sql
SELECT
  SUM (CASE WHEN exclusions != '0' AND extras != '0' THEN 1 ELSE 0 END) as both_change,
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as c
JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as r
ON c.order_id = r.order_id
WHERE cancellation = 'Successful'
```

**Answer**:

<img width="142" height="62" alt="image" src="https://github.com/user-attachments/assets/0ba7f8ce-d181-4c6e-bea4-2c2081bbdef8" />

**9. What was the total volume of pizzas ordered for each hour of the day?**
```sql
SELECT
  EXTRACT (HOUR from order_time) as hour,
  COUNT (pizza_id) as total_volume
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` 
GROUP BY hour
ORDER BY hour
```

**Answer**: The peak hours were 13:00, 18:00, 21:00, and 23:00, each recording 3 pizzas ordered. The slowest hours were 11:00 and 19:00, with only 1 pizza ordered in each.

<img width="275" height="210" alt="image" src="https://github.com/user-attachments/assets/5d1f0b50-8250-4644-aa7c-50931d6b7ed9" />

**10. What was the volume of orders for each day of the week?**
```sql
SELECT
  FORMAT_TIMESTAMP('%A', order_time) AS days_of_week,
  COUNT (pizza_id) as total_volume
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` 
GROUP BY days_of_week
ORDER BY days_of_week
```

**Answer**: The highest daily volume occurred on Wednesday and Saturday, with 5 pizzas sold on each day. The lowest volume was on Friday, with only 1 pizza sold.

<img width="296" height="151" alt="image" src="https://github.com/user-attachments/assets/c76aaa8a-3b1e-4e69-804c-49a93b18c97b" />



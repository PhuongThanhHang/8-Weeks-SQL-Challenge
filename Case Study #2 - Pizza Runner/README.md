# Case Study #2 - Pizza Runner

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

### B. Runner and Customer Experience

**1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)**

- Calculate the day difference between each registration_date and the start date 2021-01-01. Use `DATE_DIFF(end_date, start_date, DAY)`
- Divide the day difference by 7. `Using DIV(days_from_start, 7)` to determine which week each record belongs to.
- Add 1 and generate a week label to form a simple week reference table for reporting and analysis.

```sql
WITH week_reference AS (
  SELECT
    registration_date,
    DATE_DIFF(registration_date, '2021-01-01', DAY) AS days_from_Jan1,
    DIV (DATE_DIFF(registration_date, '2021-01-01', DAY), 7) AS week_index_zero,
    1 + DIV (DATE_DIFF(registration_date, '2021-01-01', DAY), 7) AS week_number
  FROM `weeksqlchallenge-474608.2pizzarunner.runners`
)

SELECT
  w.week_number,
  COUNT (runner_id) as signed_up_number
FROM `weeksqlchallenge-474608.2pizzarunner.runners` as r
JOIN week_reference as w
ON r.registration_date = w.registration_date
GROUP BY w.week_number
ORDER BY w.week_number
```

**Answer**:

<img width="277" height="120" alt="image" src="https://github.com/user-attachments/assets/95f18850-552b-4959-9d4b-8da466f1d60c" />

**2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?**

- Initially, I did it this way. **But this approach is incorrect** because after join, each pizza creates a separate row, orders with multiple pizzas get counted multiple times.
```sql
SELECT
  r.runner_id,
  ROUND (AVG(TIMESTAMP_DIFF (r.pickup_time,c.order_time,MINUTE)),2) AS difference
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` AS c
JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` AS r
ON c.order_id = r.order_id
WHERE cancellation = 'Successful'
GROUP BY r.runner_id
```

- Then, I switched to this approach. In the CTE, you apply `GROUP BY r.order_id, r.runner_id, r.pickup_time, c.order_time`, which means any rows sharing the same order_id, pickup_time, and order_time will be aggregated into a single row. This means you calculate the time for each order only once, and then compute `AVG(time_dif) by runner_id`. So the average is based on orders, not duplicated by the number of pizzas.

```sql
WITH cte AS (
  SELECT
    r.order_id,
    r.runner_id,
    TIMESTAMP_DIFF (r.pickup_time, c.order_time, MINUTE) as time_difference
  FROM `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as r
  JOIN `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as c
  ON r.order_id = c.order_id
  WHERE cancellation = 'Successful'
  GROUP BY r.order_id,r.runner_id, c.order_time, r.pickup_time
  ORDER BY r.order_id,r.runner_id
)

SELECT
  runner_id,
  ROUND (AVG (time_difference),2) as avg_time
FROM cte
GROUP BY runner_id
ORDER BY runner_id
```

**Answer**:

<img width="277" height="120" alt="image" src="https://github.com/user-attachments/assets/db4c1c45-33e8-448c-898f-6cc02a45019b" />

**3. Is there any relationship between the number of pizzas and how long the order takes to prepare?**
```sql
WITH cte AS (
  SELECT
    r.order_id,
    COUNT (c.pizza_id) as number_of_pizza,
    TIMESTAMP_DIFF (r.pickup_time, c.order_time, MINUTE) as time_difference
  FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as c
  JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as r
  ON c.order_id = r.order_id
  WHERE cancellation = 'Successful'
  GROUP BY r.order_id, c.order_time, r.pickup_time
  ORDER BY r.order_id
)

SELECT
  number_of_pizza,
  ROUND (AVG (time_difference),2) as avg_prepare_time
FROM cte
GROUP BY number_of_pizza
ORDER BY number_of_pizza
```

**Answer**: The more pizzas there are, the longer they will take to prepare.

<img width="277" height="120" alt="image" src="https://github.com/user-attachments/assets/3f95e9e0-aa61-4782-8457-0b52fb104bb0" />

**4. What was the average distance travelled for each customer?**
```sql
SELECT
  c.customer_id,
  ROUND (AVG (r.distance),2) as avg_distance
FROM `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as r
JOIN `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as c  
ON r.order_id = c.order_id
WHERE cancellation = 'Successful'
GROUP BY c.customer_id
ORDER BY c.customer_id
```

**Answer**:

<img width="278" height="178" alt="image" src="https://github.com/user-attachments/assets/cf07f87b-512f-4a8d-8f4a-f436fbd5e68e" />

**5. What was the difference between the longest and shortest delivery times for all orders?**
```sql
SELECT
  MIN (duration) as min_duration,
  MAX (duration) as max_duration,
  MAX (duration) - MIN (duration) as difference
FROM `weeksqlchallenge-474608.2pizzarunner.runner_orders_new`
WHERE cancellation = 'Successful'
```

**Answer**:

<img width="366" height="63" alt="image" src="https://github.com/user-attachments/assets/0fede9f2-72db-4e34-8d40-b50021041ed5" />

**6. What was the average speed for each runner for each delivery and do you notice any trend for these values?**
```sql
SELECT
  runner_id,
  order_id,
  distance,
  ROUND (duration/60,2) as duration_hour,
  ROUND (distance / (duration/60),2) as speed
FROM `weeksqlchallenge-474608.2pizzarunner.runner_orders_new`
WHERE cancellation = 'Successful' 
GROUP BY order_id, runner_id, distance, duration
ORDER BY runner_id,order_id
```

**Answer**: The running speed of each runner increases progressively with each subsequent delivery.

<img width="516" height="267" alt="image" src="https://github.com/user-attachments/assets/be145589-472c-4576-b812-562913ce3580" />

**7. What is the successful delivery percentage for each runner?**

```sql
SELECT 
  runner_id,
  SUM(CASE WHEN cancellation = 'Successful' THEN 1 ELSE 0 END) / COUNT(*) * 100 AS successful_delivery_percentage
FROM `weeksqlchallenge-474608.2pizzarunner.runner_orders_new`
GROUP BY runner_id
ORDER BY successful_delivery_percentage
```

**Answer**:

<img width="321" height="122" alt="image" src="https://github.com/user-attachments/assets/c7ea56a6-2e34-49bd-9ec3-9074616327fa" />

### C. Ingredient Optimisation

**1. What are the standard ingredients for each pizza?**

First, I create cte1 to split toppings in a string into separate rows using the following steps:
- Use SPLIT(toppings, ',') to convert the string into an array
- Apply UNNEST(...) to expand the array into multiple rows
- Use TRIM() to remove extra spaces and CAST(... AS INT64) to convert the text values into integers

Then, I join the pizza_toppings table with cte1 to retrieve the topping_name for each pizza_id.

Finally, because the pizza_id and topping_name table is too long to read, I combine all topping_name values for each pizza_id into a single string to make it easier to read.

```sql
WITH cte1 AS (
  SELECT
    pizza_id,
    CAST (TRIM (toppings) AS INT64) as topping_id
  FROM `weeksqlchallenge-474608.2pizzarunner.pizza_recipes`,
  UNNEST (SPLIT (toppings, ',')) AS toppings
),

cte2 AS (
  SELECT
    c.pizza_id,
    p.topping_name
  FROM cte1 as c
  JOIN `weeksqlchallenge-474608.2pizzarunner.pizza_toppings` as p
  ON c.topping_id = p.topping_id
)

SELECT
  pizza_id,
  STRING_AGG(topping_name, ', ') as standard_toppings
FROM cte2
GROUP BY pizza_id
ORDER BY pizza_id
```

**Answer**:

<img width="581" height="86" alt="image" src="https://github.com/user-attachments/assets/da6a61fb-8072-4021-9640-9cd25d5f773d" />

**2. What was the most commonly added extra?**

- First, create cte1 to split the strings in the extras column into separate rows by using `SPLIT (colume_name, ',')` và `UNNEST () AS`
- Then join cte1 with the pizza_toppings table to retrieve the corresponding topping_name.
- Finally, calculate the total number of times each topping was added.

```sql
WITH cte1 AS (
  SELECT
    order_id,
    pizza_id,
    CAST (TRIM (extras) AS INT64) AS extras
  FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new`,
  UNNEST (SPLIT (extras, ',')) as extras
)

SELECT
 p.topping_name,
 COUNT (extras) as number
FROM cte1 as c
JOIN `weeksqlchallenge-474608.2pizzarunner.pizza_toppings` as p
ON c.extras = p.topping_id
WHERE extras != 0
GROUP BY p.topping_name
ORDER BY number DESC
```

**Answer**: The most commonly added extra is Bacon.

<img width="360" height="115" alt="image" src="https://github.com/user-attachments/assets/a84f079e-f43f-4f65-9f81-e6894fcb981f" />

**3. What was the most common exclusion?**
```sql
WITH cte1 AS (
  SELECT
    order_id,
    pizza_id,
    CAST (TRIM (exclusions) AS INT64) AS exclusions
  FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new`,
  UNNEST (SPLIT (exclusions, ',')) as exclusions
)

SELECT
 p.topping_name,
 COUNT (exclusions) as number
FROM cte1 as c
JOIN `weeksqlchallenge-474608.2pizzarunner.pizza_toppings` as p
ON c.exclusions = p.topping_id
WHERE exclusions != 0
GROUP BY p.topping_name
ORDER BY number DESC
```

**Answer**: The most common exclusion is Cheese.

<img width="273" height="118" alt="image" src="https://github.com/user-attachments/assets/566974da-19c3-4d4b-8677-61c545e2cdf1" />

**6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?**

I solved this question using two methods, both based on the same idea. The second method is simply a more concise version of the first one.

**Idea**: Create 3 tables 
- Table 1 – Standard_toppings: this table calculates the total number of standard toppings used across all successfully delivered pizzas. It contains two columns: topping_id and standard_count.
- Table 2 – Extras: this table calculates the total number of extra toppings added across all successfully delivered pizzas. It contains two columns: topping_id and extras_count.
- Table 3 – Exclusions: this table calculates the total number of toppings removed across all successfully delivered pizzas. It contains two columns: topping_id and exclusions_count.
- Next, perform a LEFT JOIN of Table 2 and Table 3 onto Table 1. This produces a consolidated table named JOIN_table, containing four columns: topping_id, standard_count, extras_count, and exclusions_count.
- From the JOIN_table, calculate the actual number of toppings used across all pizzas using the formula:
`standard_count + extras_count − exclusions_count`. This produces the final result table with two columns: topping_id and total_quantity.

**Method 1**

- First, extract a table containing the key information for all pizzas that were successfully delivered.
- Create Table 1 – standard_toppings using two steps: Create cte1 to split the topping strings into separate rows. Then, generate Standard_toppings by applying COUNT and GROUP BY topping_id on the cte1 table.
- Do the same for the Extras and Exclusions tables
- Next, LEFT JOIN Extras and Exclusions tables onto standard_toppings. This produces a consolidated table named join_table, containing four columns: topping_id, standard_count, extras_count, and exclusions_count. 
- From the join_table, using the formula: `standard_count + COALESCE(extras_count,0) - COALESCE(exclusions_count,0)`. Using `COALESCE(column_name, 0)` converts any NULL values in the column to 0, allowing BigQuery to perform calculations without errors.

```sql
WITH delivered_pizzas AS (
  SELECT
    c.pizza_id,
    p.toppings,
    c.exclusions,
    c.extras
  FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as c
  JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as r
  ON c.order_id = r.order_id
  JOIN `weeksqlchallenge-474608.2pizzarunner.pizza_recipes` as p
  ON c.pizza_id = p.pizza_id
  WHERE cancellation = 'Successful'
), 

standard_toppings AS (
SELECT
  CAST(TRIM(toppings) AS INT64) AS topping_id,
  COUNT(*) AS standard_count
FROM delivered_pizzas,
UNNEST(SPLIT(toppings, ',')) AS toppings
WHERE TRIM(toppings) != '0'
GROUP BY topping_id
),

exclusions AS (
SELECT 
  CAST (TRIM (exclusions) AS INT64) as topping_id,
  COUNT (*) as exclusions_count
FROM delivered_pizzas,
UNNEST (SPLIT (exclusions, ',')) AS exclusions
WHERE TRIM (exclusions) != '0'
GROUP BY topping_id
),

extras AS (
SELECT 
  CAST (TRIM (extras) AS INT64) as topping_id,
  COUNT (*) as extras_count
FROM delivered_pizzas,
UNNEST (SPLIT (extras, ',')) AS extras
WHERE TRIM (extras) != '0'
GROUP BY topping_id
),

join_table AS (
SELECT
  s.topping_id,
  s.standard_count,
  et.extras_count,
  ec.exclusions_count
FROM standard_toppings as s
LEFT JOIN extras as et ON s.topping_id = et.topping_id
LEFT JOIN exclusions as ec ON s.topping_id = ec.topping_id
)

SELECT 
  p.topping_name,
  standard_count + COALESCE(extras_count,0) - COALESCE(exclusions_count,0) AS total_quantity
FROM join_table as j
JOIN `weeksqlchallenge-474608.2pizzarunner.pizza_toppings` as p
ON j.topping_id = p.topping_id
ORDER BY total_quantity DESC
```

**Answer**:

**Method 2**:

- First, extract a table containing the key information for all pizzas that were successfully delivered (Same as in Method 1).
- Instead of using six CTEs (cte1, standard_topping, cte2, extras, cte3, exclusions) and joining them, I create a single CTE called topping_event that works as follows:
UNNEST the standard_toppings and assign a +1 flag
UNNEST the extras and assign a +1 flag
UNNEST the exclusions and assign a -1 flag
To simplify the calculation of total toppings used in delivered pizzas, we convert each topping event into a unified form called delta, which represents how each row should adjust the total count.
- Use `UNION ALL` to combine all these unnests into a single event row for each topping. Then, simply SUM(delta) grouped by topping_id to get the final result: standard + extras − exclusions in a single step.

```sql
WITH delivered_pizzas AS ( 
  SELECT
    c.pizza_id,
    p.toppings,
    c.exclusions,
    c.extras
  FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` AS c
  JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` AS r
    ON c.order_id = r.order_id
  JOIN `weeksqlchallenge-474608.2pizzarunner.pizza_recipes` AS p
    ON c.pizza_id = p.pizza_id
  WHERE r.cancellation = 'Successful'
),

topping_events AS (
  -- standard toppings: +1
  SELECT 
    CAST(TRIM(toppings) AS INT64) AS topping_id,
    1 AS delta
  FROM delivered_pizzas,
  UNNEST(SPLIT(toppings, ',')) AS toppings
  WHERE TRIM(toppings) != '0'

  UNION ALL

  -- extras: +1
  SELECT 
    CAST(TRIM(extras) AS INT64) AS topping_id,
    1 AS delta
  FROM delivered_pizzas,
  UNNEST(SPLIT(extras, ',')) AS extras
  WHERE TRIM(extras) != '0'

  UNION ALL

  -- exclusions: -1
  SELECT 
    CAST(TRIM(exclusions) AS INT64) AS topping_id,
    -1 AS delta
  FROM delivered_pizzas,
  UNNEST(SPLIT(exclusions, ',')) AS exclusions
  WHERE TRIM(exclusions) != '0'
)

SELECT
 topping_name,
 SUM (delta) as total_quantity
FROM topping_events as t
JOIN `weeksqlchallenge-474608.2pizzarunner.pizza_toppings` AS p
ON t.topping_id = p.topping_id 
GROUP BY topping_name
ORDER BY total_quantity DESC
```
**Answer**: Bacon, mushroom, and cheese are the three most frequently used toppings.

<img width="428" height="387" alt="image" src="https://github.com/user-attachments/assets/28df27e2-85da-4843-9258-cb9a5ff82be7" />

### D. Pricing and Ratings

**1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?**

```sql
WITH cte AS (
SELECT
  pizza_id,
  (CASE WHEN pizza_id = 1 then 12
        WHEN pizza_id = 2 then 10
        ELSE 0 END) AS price
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as c
JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as r
ON c.order_id = r.order_id
WHERE cancellation = 'Successful'
)
SELECT
  SUM (price) as total_revenue
FROM cte
```
**Answer**: Pizza Runner has made 138 dollar

<img width="108" height="49" alt="image" src="https://github.com/user-attachments/assets/4572eb4e-91e7-4387-900a-44c836299f07" />

**2. What if there was an additional $1 charge for any pizza extras? Add cheese is $1 extra**

```sql
WITH delivered_pizzas AS (
  SELECT
    c.order_id,
    c.pizza_id,
    c.extras
  FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` AS c
  JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` AS r
    ON c.order_id = r.order_id
  WHERE r.cancellation = 'Successful'
),
extras_count AS (
  SELECT 
    order_id,
    COUNT(*) AS extra_toppings
  FROM delivered_pizzas,
  UNNEST(SPLIT(extras, ',')) AS extras
  WHERE TRIM(extras) != '0'
  GROUP BY order_id
)

SELECT
  SUM(
    CASE 
      WHEN pizza_id = 1 THEN 12   
      WHEN pizza_id = 2 THEN 10   
    END
    + COALESCE(extra_toppings, 0) * 1 
  ) AS total_revenue
FROM delivered_pizzas as d
LEFT JOIN extras_count as e
ON d.order_id = e.order_id
```
**Answer**:

<img width="106" height="46" alt="image" src="https://github.com/user-attachments/assets/4b0ebffb-1918-4727-bd63-a96703c730b1" />

**3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.**

```sql
CREATE TABLE `weeksqlchallenge-474608.2pizzarunner.runner_ratings` (
  order_id INT64,
  rating INT64
);
INSERT INTO `weeksqlchallenge-474608.2pizzarunner.runner_ratings` (order_id, rating)
VALUES
  (1, 5),
  (2, 4),
  (3, 5),
  (4, 3),
  (5, 4),
  (7, 5),
  (8, 4),
  (10, 3)
```
**Answer**:

<img width="276" height="268" alt="image" src="https://github.com/user-attachments/assets/9781603f-190b-461d-a0fc-d0f76f52c45e" />

**4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?**

```sql
SELECT
  a.customer_id,
  a.order_id,
  b.runner_id,
  c.rating,
  a.order_time,
  b.pickup_time,
  TIMESTAMP_DIFF(b.pickup_time, a.order_time, MINUTE) as time_between_order_and_pickup,
  b.duration,
  ROUND(b.distance/(b.duration/60),2) as average_speed,
  COUNT (a.pizza_id) as total_pizza
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as a
JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as b
ON a.order_id = b.order_id
JOIN `weeksqlchallenge-474608.2pizzarunner.runner_ratings` as c
ON a.order_id = c.order_id
WHERE cancellation = 'Successful'
GROUP BY a.customer_id,a.order_id,b.runner_id,c.rating,a.order_time,b.pickup_time, b.duration, b.distance
ORDER BY a.customer_id,a.order_id,b.runner_id
```
**Answer**:

<img width="1158" height="269" alt="image" src="https://github.com/user-attachments/assets/abe902fe-4cb2-4330-9263-f6cc080c98cf" />

**5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?**

```sql
WITH deliverd_pizzas AS (
SELECT 
  c.order_id,
  SUM (CASE WHEN pizza_id = 1 THEN 12
       WHEN pizza_id = 2 THEN 10
       END) AS total_price,
  distance,
  distance * 0.3 AS delivery_cost
FROM `weeksqlchallenge-474608.2pizzarunner.customer_orders_new` as c
JOIN `weeksqlchallenge-474608.2pizzarunner.runner_orders_new` as r
ON c.order_id = r.order_id
WHERE cancellation = 'Successful'
GROUP BY c.order_id, distance
ORDER BY c.order_id
)

SELECT 
  SUM (total_price) as total_price,
  SUM (delivery_cost) as total_delivery_cost,
  SUM (total_price) - SUM (delivery_cost) as remaining_amount
FROM deliverd_pizzas
```
**Answer**:

<img width="414" height="61" alt="image" src="https://github.com/user-attachments/assets/30b7bfce-274e-40a1-9d29-4b7e620bd178" />

### E. Bonus Questions

**Question**:

If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?

**Answer**:

If Danny decides to expand the menu by adding a new pizza (for example, a Supreme pizza), the data design would be affected as follows:

- `pizza_names` table and `pizza_recipes` table will change, because a new record must be inserted to represent the new pizza.
- Tables like orders, customer_orders, runner_orders, pizza_toppings remain unchange. These tables are only affected when a customer actually places an order for the new pizza, at which point new rows will be inserted automatically through normal operations.
- The current schema does not require any changes because it already supports menu expansion.

```sql
INSERT INTO pizza_runner.pizza_names (pizza_id, pizza_name)
VALUES (3, 'Supreme');
INSERT INTO pizza_runner.pizza_recipes (pizza_id, toppings)
VALUES (3, '1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12');
```

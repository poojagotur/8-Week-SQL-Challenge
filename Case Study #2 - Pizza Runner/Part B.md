# 🍕 Case Study #2 Pizza Runner

## Solution - B. Runner and Customer Experience
-- How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
````sql
select date(date_trunc('week',registration_date)) + 4 as week_start,
count(runner_id)
from pizza_runner.runners
group by week_start
````

-- What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?

````sql
select runner_id, avg(arrive_time) from
(
select *,
date_trunc('min',pickup_time::timestamp)-date_trunc('min',order_time) as arrive_time
from pizza_runner.customer_orders
join pizza_runner.runner_orders using(order_id)
where pickup_time<>'null'
)as t
group by runner_id
````

Is there any relationship between the number of pizzas and how long the order takes to prepare?

````sql
SELECT 
  pizza_order, 
  AVG(prep_time_minutes) AS avg_prep_time_minutes from 
(
   SELECT 
    c.order_id, 
    COUNT(c.order_id) AS pizza_order,
    EXTRACT(EPOCH FROM (r.pickup_time::timestamp - c.order_time::timestamp)) / 60 AS prep_time_minutes
FROM  pizza_runner.customer_orders AS c
JOIN pizza_runner.runner_orders AS r ON c.order_id = r.order_id
WHERE pickup_time<>'null'
GROUP BY c.order_id, c.order_time, r.pickup_time
)as t
WHERE prep_time_minutes > 1
GROUP BY pizza_order;
````

What was the average distance traveled for each customer?
````sql
select customer_id,
avg(replace(distance,'km','')::decimal)
from pizza_runner.runner_orders
join pizza_runner.customer_orders using(order_id)
where distance<>'null'
group by customer_id
````

-- What was the average distance travelled for each customer?
````sql
select customer_id,
avg(replace(distance,'km','')::decimal)
from pizza_runner.runner_orders
join pizza_runner.customer_orders using(order_id)
where distance<>'null'
group by customer_id
````

Removing : mins, minutes: REGEXP_REPLACE(duration, '[^0-9]', '', 'g')
````sql
-- What was the difference between the longest and shortest delivery times for all orders?
SELECT max(duration_changed)::integer- min(duration_changed)::integer as difference from(
select
*, REGEXP_REPLACE(duration, '[^0-9]', '', 'g') as duration_changed
FROM pizza_runner.runner_orders
where pickup_time<>'null'
) as t
````

-- What was the average speed for each runner for each delivery and do you notice any trend for these values?

````sql
select runner_id,order_id, dist/tim*60 as speed from(
select
*, 
replace(distance, 'km',''):: decimal as dist ,
regexp_replace(duration,'[^0-9]','','g'):: decimal as tim 
from pizza_runner.runner_orders
where pickup_time<>'null'
)
as t 
order by 1, 2
````


-- -- What is the successful delivery percentage for each runner?
````sql
select runner_id, round((success/total)*100,2) from
(
select 
runner_id, sum(case when pickup_time<>'null' then 1 else 0 end)::decimal as success, count(*)::decimal as total
from pizza_runner.runner_orders
group by runner_id
) as t
````


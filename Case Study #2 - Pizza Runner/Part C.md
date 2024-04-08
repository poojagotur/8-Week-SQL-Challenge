# 🍕 Case Study #2 Pizza Runner

## Solution - C. Ingredient Optimisation

### 1. What are the standard ingredients for each pizza?
Basically find the toppings that are present as base for all pizzas, by finding the count of toppings for each pizza.
Need to find the pizza names and the corresponding topping names
Find the topping id from toppings in pizza_recipe table
````sql
SELECT 
pizza_id, unnest(string_to_array(toppings, ', ')) as topping_id
FROM pizza_runner.pizza_recipes
````
Final Solution:
````sql
select topping_name, count(topping_name) from(
SELECT 
pizza_id, unnest(string_to_array(toppings, ', ')):: integer as topping_id
FROM pizza_runner.pizza_recipes
  ) as t
join pizza_runner.pizza_toppings using(topping_id)
group by 1
having count(topping_name) >1
````
### 2. What was the most commonly added extra?
````sql
select pt.topping_name, count(t.extra_topping) from(
select 
unnest(string_to_array(extras, ', '))::integer as extra_topping
from pizza_runner.customer_orders 
where extras<>'null' and extras<>''
)as t
join pizza_runner.pizza_toppings pt on t.extra_topping=pt.topping_id
group by 1
order by 2 desc 
limit 1
````
### 3. What was the most common exclusion?
````sql
select pt.topping_name, count(t.extra_topping) from(
select 
unnest(string_to_array(exclusions, ', '))::integer as extra_topping
from pizza_runner.customer_orders 
where exclusions<>'null' and exclusions<>''
)as t
join pizza_runner.pizza_toppings pt on t.extra_topping=pt.topping_id
group by 1
order by 2 desc 
limit 1
````
### 4. -- Generate an order item for each record in the customers_orders table in the format of one of the following:
-- Meat Lovers
-- Meat Lovers - Exclude Beef
-- Meat Lovers - Extra Bacon
-- Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers

````sql
WITH EXCLUSIONS AS (
    SELECT 
    order_id,
    pizza_id,
    S.value as topping_id,
    T.topping_name
    FROM customer_orders as co
    LEFT JOIN LATERAL SPLIT_TO_TABLE(exclusions,', ') as S
    INNER JOIN pizza_toppings as T on t.topping_id = S.value
    WHERE LENGTH(value)>0 AND value<>'null'
)
,EXTRAS AS (
    SELECT 
    order_id,
    pizza_id,
    S.value as topping_id,
    T.topping_name
    FROM customer_orders as co
    LEFT JOIN LATERAL SPLIT_TO_TABLE(extras,', ') as S
    INNER JOIN pizza_toppings as T on t.topping_id = S.value
    WHERE LENGTH(value)>0 AND value<>'null'
)
,ORDERS AS (
    SELECT DISTINCT
    CO.order_id,
    CO.pizza_id,
    S.value as topping_id
    FROM customer_orders as CO
    INNER JOIN pizza_recipes as PR on CO.pizza_id = PR.pizza_id
    LEFT JOIN LATERAL SPLIT_TO_TABLE(toppings,', ') as S
)
,ORDERS_WITH_EXTRAS_AND_EXCLUSIONS AS (
    SELECT
    O.order_id,
    O.pizza_id,
    CASE 
    WHEN O.pizza_id = 1 THEN 'Meat Lovers'
    WHEN O.pizza_id = 2 THEN pizza_name
    END as pizza, 
    LISTAGG(DISTINCT EXT.topping_name, ', ') as extras,
    LISTAGG(DISTINCT EXC.topping_name, ', ') as exclusions
    FROM ORDERS AS O
    LEFT JOIN EXTRAS AS EXT ON EXT.order_id=O.order_id AND EXT.pizza_id=O.pizza_id
    LEFT JOIN EXCLUSIONS AS EXC ON EXC.order_id=O.order_id AND EXC.pizza_id=O.pizza_id AND EXC.topping_id=O.topping_id 
    INNER JOIN pizza_names as PN on O.pizza_id = PN.pizza_id
    GROUP BY O.order_id,
    O.pizza_id,
    CASE 
    WHEN O.pizza_id = 1 THEN 'Meat Lovers'
    WHEN O.pizza_id = 2 THEN pizza_name
    END
)

SELECT 
order_id,
pizza_id,
CONCAT(pizza, 
CASE WHEN exclusions = '' THEN '' ELSE ' - Exclude ' || exclusions END,
CASE WHEN extras = '' THEN '' ELSE ' - Extra ' || extras END) as order_item
FROM ORDERS_WITH_EXTRAS_AND_EXCLUSIONS
ORDER BY order_id; 
````
###-- 5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
-- For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
````sql
WITH EXCLUSIONS AS (
    SELECT 
    order_id,
    pizza_id,
    S.value as topping_id
    FROM customer_orders as co
    LEFT JOIN LATERAL SPLIT_TO_TABLE(exclusions,', ') as S
    WHERE LENGTH(value)>0 AND value<>'null'
)
,EXTRAS AS (
    SELECT 
    order_id,
    pizza_id,
    S.value as topping_id,
    topping_name
    FROM customer_orders as co
    LEFT JOIN LATERAL SPLIT_TO_TABLE(extras,', ') as S
    INNER JOIN pizza_toppings as T on t.topping_id = S.value
    WHERE LENGTH(value)>0 AND value<>'null'
)
,ORDERS AS (
    SELECT DISTINCT
    CO.order_id,
    CO.pizza_id,
    S.value as topping_id,
    topping_name
    FROM customer_orders as CO
    INNER JOIN pizza_recipes as PR on CO.pizza_id = PR.pizza_id
    LEFT JOIN LATERAL SPLIT_TO_TABLE(toppings,', ') as S
    INNER JOIN pizza_toppings as T on t.topping_id = S.value
)
,ORDERS_WITH_EXTRAS_AND_EXCLUSIONS AS (
    SELECT 
    O.order_id,
    O.pizza_id,
    O.topping_id::int as topping_id,
    topping_name
    FROM ORDERS AS O
    LEFT JOIN EXCLUSIONS AS EXC ON EXC.order_id=O.order_id AND EXC.pizza_id=O.pizza_id AND EXC.topping_id=O.topping_id 
    WHERE EXC.topping_id IS NULL

    UNION ALL 

    SELECT 
    order_id,
    pizza_id,
    topping_id::int as topping_id,
    topping_name
    FROM EXTRAS
    WHERE topping_id<>''
)
,TOPPING_COUNT AS (
    SELECT 
    O.order_id,
    O.pizza_id,
    O.topping_name,
    COUNT(*) as n
    FROM ORDERS_WITH_EXTRAS_AND_EXCLUSIONS as O
    GROUP BY 
    O.order_id,
    O.pizza_id,
    O.topping_name
)
SELECT 
order_id,
pizza_id,
LISTAGG(
CASE
    WHEN n>1 THEN n || 'x' || topping_name
    ELSE topping_name
END,', ') as ingredient
FROM TOPPING_COUNT
GROUP BY order_id,
pizza_id;
````
-- 6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?
````sql
WITH EXCLUSIONS AS (
    SELECT 
    order_id,
    pizza_id,
    S.value as topping_id
    FROM customer_orders as co
    LEFT JOIN LATERAL SPLIT_TO_TABLE(exclusions,', ') as S
    WHERE LENGTH(value)>0 AND value<>'null'
)
,EXTRAS AS (
    SELECT 
    order_id,
    pizza_id,
    S.value as topping_id,
    topping_name
    FROM customer_orders as co
    LEFT JOIN LATERAL SPLIT_TO_TABLE(extras,', ') as S
    INNER JOIN pizza_toppings as T on t.topping_id = S.value
    WHERE LENGTH(value)>0 AND value<>'null'
)
,ORDERS AS (
    SELECT DISTINCT
    CO.order_id,
    CO.pizza_id,
    S.value as topping_id,
    topping_name
    FROM customer_orders as CO
    INNER JOIN pizza_recipes as PR on CO.pizza_id = PR.pizza_id
    LEFT JOIN LATERAL SPLIT_TO_TABLE(toppings,', ') as S
    INNER JOIN pizza_toppings as T on t.topping_id = S.value
)
,ORDERS_WITH_EXTRAS_AND_EXCLUSIONS AS (
    SELECT 
    O.order_id,
    O.pizza_id,
    O.topping_id::int as topping_id,
    topping_name
    FROM ORDERS AS O
    LEFT JOIN EXCLUSIONS AS EXC ON EXC.order_id=O.order_id AND EXC.pizza_id=O.pizza_id AND EXC.topping_id=O.topping_id 
    WHERE EXC.topping_id IS NULL

    UNION ALL 

    SELECT 
    order_id,
    pizza_id,
    topping_id::int as topping_id,
    topping_name
    FROM EXTRAS
    WHERE topping_id<>''
)

SELECT 
O.topping_name,
COUNT(O.pizza_id) as ingredient_count
FROM ORDERS_WITH_EXTRAS_AND_EXCLUSIONS as O
INNER JOIN runner_orders as ro on O.order_id = ro.order_id
WHERE pickup_time<>'null'
GROUP BY 
O.topping_name
ORDER BY COUNT(O.pizza_id) DESC

````






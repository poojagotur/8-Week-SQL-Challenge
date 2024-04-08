# 🍕 Case Study #2 Pizza Runner

## Solution - C. Ingredient Optimisation

### 1. What are the standard ingredients for each pizza?
Need to find the pizza names and the corresponding topping names
Find the topping id from toppings in pizza_recipe table
````sql
SELECT 
pizza_id, unnest(string_to_array(toppings, ', ')) as topping_id
FROM pizza_runner.pizza_recipes
````

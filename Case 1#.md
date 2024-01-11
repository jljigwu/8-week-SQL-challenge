# Case Study #1 - Danny's Diner 

## Business Objective 
> Danny, the restaurant manager, is driven by the desire to understand his customers better. By leveraging data analytics, he aims to answer key questions about their visiting patterns, expenditure patterns, and preferred menu items. This strategic approach allows Danny to tailor the restaurant experience, optimize offerings, and enhance customer satisfaction.

1. What is the total amount each customer spent at the restaurant?
   
```sql
SELECT
    sales.customer_id,
    SUM(menu.price) AS total_sales
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY total_sales DESC;
```
2. How many days has each customer visited the restaurant?
```sql
SELECT
  	customer_id,
   	COUNT( DISTINCT order_date) as visited_times -- avoid duplicate counting of days
FROM dannys_diner.sales 
GROUP BY customer_id
ORDER BY visited_times DESC; 
```
3. What was the first item from the menu purchased by each customer?
```sql
WITH ranked_sales AS
(
SELECT
  	sales.customer_id,
    menu.product_name,
    sales.order_date,
  	RANK() 
  	OVER ( PARTITION BY sales.customer_id ORDER BY sales.order_date) AS rank
FROM dannys_diner.sales
JOIN dannys_diner.menu ON sales.product_id = menu.product_id
)

SELECT 
	customer_id,
    product_name
FROM ranked_sales
WHERE rank = 1
GROUP BY customer_id, product_name;
 ```
4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT 
	menu.product_name,
    COUNT(sales.product_id) as counted_sales
FROM dannys_diner.menu
JOIN dannys_diner.sales ON menu.product_id = sales.product_id
GROUP BY product_name
LIMIT 1;
```


5. Which item was the most popular for each customer?
```sql
WITH ranked_product AS
(
 SELECT
 	menu.product_name,
    sales.customer_id,
    COUNT(menu.product_id) AS order_count,
    RANK() 
  	OVER(PARTITION BY sales.customer_id ORDER BY COUNT(sales.customer_id) ASC)
     AS rank
 FROM dannys_diner.sales
 JOIN dannys_diner.menu on sales.product_id = menu.product_id
 GROUP BY sales.customer_id, menu.product_id, menu.product_name
  )
 SELECT 
  customer_id, 
  product_name, 
  order_count
  FROM ranked_product 
  WHERE rank = 1;
```
6. Which item was purchased first by the customer after they became a member?
```sql
WITH first_order AS
(
  SELECT
    members.customer_id,
    members.join_date,
    sales.order_date,
    sales.product_id,
  	menu.product_name,
    ROW_NUMBER() OVER (PARTITION BY members.customer_id ORDER BY sales.order_date) AS rank
  FROM dannys_diner.members
  JOIN dannys_diner.sales ON members.customer_id = sales.customer_id
  JOIN dannys_diner.menu ON sales.product_id = menu.product_id
  WHERE sales.order_date > members.join_date
)
SELECT
  first_order.customer_id,
  first_order.join_date,
  first_order.order_date,
  first_order.product_name
FROM first_order
WHERE rank = 1;
```
7. Which item was purchased just before the customer became a member?
```sql
WITH first_order AS
(
  SELECT
    members.customer_id,
    members.join_date,
    sales.order_date,
    sales.product_id,
  	menu.product_name,
    RANK() OVER (PARTITION BY members.customer_id ORDER BY sales.order_date DESC) AS rank
  FROM dannys_diner.members
  JOIN dannys_diner.sales ON members.customer_id = sales.customer_id
  JOIN dannys_diner.menu ON sales.product_id = menu.product_id
  WHERE sales.order_date < members.join_date ORDER BY sales.order_date 
)
SELECT
  first_order.customer_id,
  first_order.join_date,
  first_order.order_date,
  first_order.product_name
FROM first_order
WHERE rank = 1;
```
8. What is the total items and amount spent for each member before they became a member?
```sql
SELECT 
  sales.customer_id, 
  COUNT(sales.product_id) AS total_items, 
  SUM(menu.price) AS total_sales
FROM dannys_diner.sales
JOIN dannys_diner.members
  ON sales.customer_id = members.customer_id
  AND sales.order_date < members.join_date
JOIN dannys_diner.menu
  ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```


9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
WITH points AS (
  SELECT 
    menu.product_id, 
    CASE
      WHEN product_id = 1 THEN price * 20
      ELSE price * 10 END AS points
  FROM dannys_diner.menu
)

SELECT 
  sales.customer_id, 
  SUM(points.points) AS total_points
FROM dannys_diner.sales
INNER JOIN points
  ON sales.product_id = points.product_id
GROUP BY sales.customer_id
ORDER BY total_points DESC;

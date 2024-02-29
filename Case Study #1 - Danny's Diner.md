# Case Study #1 - Danny's Diner 

## Business Objective 
> Danny, the restaurant manager, is driven by the desire to understand his customers better. By leveraging data analytics, he aims to answer key questions about their visiting patterns, expenditure patterns, and preferred menu items. This strategic approach allows Danny to tailor the restaurant experience, optimize offerings, and enhance customer satisfaction.

## Relationship Diagram
![image](https://github.com/jljigwu/8-week-SQL-challenge/assets/107698857/37e61127-42aa-43f9-a367-1af4ae960b96)
---

## Schema (PostgreSQL v13)
```
CREATE SCHEMA dannys_diner;
    SET search_path = dannys_diner;
    CREATE TABLE sales (
      "customer_id" VARCHAR(1),
      "order_date" DATE,
      "product_id" INTEGER
    );
    INSERT INTO sales
      ("customer_id", "order_date", "product_id")
    VALUES
      ('A', '2021-01-01', '1'),
      ('A', '2021-01-01', '2'),
      ('A', '2021-01-07', '2'),
      ('A', '2021-01-10', '3'),
      ('A', '2021-01-11', '3'),
      ('A', '2021-01-11', '3'),
      ('B', '2021-01-01', '2'),
      ('B', '2021-01-02', '2'),
      ('B', '2021-01-04', '1'),
      ('B', '2021-01-11', '1'),
      ('B', '2021-01-16', '3'),
      ('B', '2021-02-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-07', '3');
     
    
    CREATE TABLE menu (
      "product_id" INTEGER,
      "product_name" VARCHAR(5),
      "price" INTEGER
    );
    
    INSERT INTO menu
      ("product_id", "product_name", "price")
    VALUES
      ('1', 'sushi', '10'),
      ('2', 'curry', '15'),
      ('3', 'ramen', '12');
      
    
    CREATE TABLE members (
      "customer_id" VARCHAR(1),
      "join_date" DATE
    );
    
    INSERT INTO members
      ("customer_id", "join_date")
    VALUES
      ('A', '2021-01-07'),
      ('B', '2021-01-09');

```
---
**Query #1**
What is the total amount each customer spent at the restaurant?
   
```sql
SELECT
    sales.customer_id,
    SUM(menu.price) AS total_sales
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY total_sales DESC;
```
Answer:
| customer_id | total_sales |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |



---

**Query #2**
How many days has each customer visited the restaurant?
```sql
SELECT
   customer_id,
   COUNT( DISTINCT order_date) as visited_times -- avoid duplicate counting of days
FROM dannys_diner.sales 
GROUP BY customer_id
ORDER BY visited_times DESC; 
```
Answer: 
| customer_id | visited_times |
| ----------- | ------------- |
| B           | 6             |
| A           | 4             |
| C           | 2             |

---

**Query #3**
What was the first item from the menu purchased by each customer?
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
Answer: 
| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

---

**Query #4**
What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT
    menu.product_name,
    COUNT(sales.product_id) as counted_sales
FROM dannys_diner.menu
JOIN dannys_diner.sales ON menu.product_id = sales.product_id
GROUP BY product_name
order by counted_sales DESC
LIMIT 1;
```
Answer:
| product_name | counted_sales |
| ------------ | ------------- |
| ramen        | 8             |

---

**Query #5**
 Which item was the most popular for each customer?
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
 Answer:
| customer_id | product_name | order_count |
| ----------- | ------------ | ----------- |
| A           | sushi        | 1           |
| B           | ramen        | 2           |
| B           | sushi        | 2           |
| B           | curry        | 2           |
| C           | ramen        | 3           |

---

**Query #6**
 Which item was purchased first by the customer after they became a member?
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
Answer:

| customer_id | join_date                | order_date               | product_name |
| ----------- | ------------------------ | ------------------------ | ------------ |
| A           | 2021-01-07T00:00:00.000Z | 2021-01-10T00:00:00.000Z | ramen        |
| B           | 2021-01-09T00:00:00.000Z | 2021-01-11T00:00:00.000Z | sushi        |

---

**Query #7**
 Which item was purchased just before the customer became a member?
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
Answer:

| customer_id | join_date                | order_date               | product_name |
| ----------- | ------------------------ | ------------------------ | ------------ |
| A           | 2021-01-07T00:00:00.000Z | 2021-01-01T00:00:00.000Z | sushi        |
| A           | 2021-01-07T00:00:00.000Z | 2021-01-01T00:00:00.000Z | curry        |
| B           | 2021-01-09T00:00:00.000Z | 2021-01-04T00:00:00.000Z | sushi        |

---

**Query #8**
What is the total items and amount spent for each member before they became a member?
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
Answer:
| customer_id | total_items | total_sales |
| ----------- | ----------- | ----------- |
| A           | 2           | 25          |
| B           | 3           | 40          |

---

**Query #9** If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
WITH points AS
(
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
```
Answer:
| customer_id | total_points |
| ----------- | ------------ |
| B           | 940          |
| A           | 860          |
| C           | 360          |

---

**Query #10**
 In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi — how many points do customer A and B have at the end of January?
 ```sql
WITH promo AS 
(
SELECT *
	  , (join_date + INTERVAL '6 days')::date as promo
	  , (DATE_TRUNC('month', join_date) + INTERVAL '1 month - 1 day')::date as time
FROM dannys_diner.members m
)

SELECT 
	customer_id AS customer,
    SUM(CASE
		WHEN  product_name = 'sushi'
		 THEN price * 20
		WHEN order_date BETWEEN join_date AND promo
		 THEN price * 20
		ELSE price * 10
	 END) points
FROM promo p
JOIN dannys_diner.sales s
USING(customer_id)
JOIN dannys_diner.menu m
USING(product_id)
WHERE order_date <= time
GROUP BY customer_id

```

Answer:

| customer_id | total_points |
| ----------- | ------------ |
| B           | 820          |
| A           | 1370         |

---

**Bonus Query #1** Join All The Things - Recreate the following table output using the available data:

| customer_id | order_date               | product_name | price | member |
| ----------- | ------------------------ | ------------ | ----- | ------ |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      |

``` sql
SELECT 
  s.customer_id,
  s.order_date,
  m.product_name,
  m.price,
  CASE 
    WHEN order_date < join_date OR join_date is null
      THEN 'N'
    ELSE 'Y'
  END AS member
FROM 
  dannys_diner.sales s
  LEFT JOIN dannys_diner.members mem
    USING(customer_id)
  LEFT JOIN dannys_diner.menu m
    USING (product_id)
ORDER BY order_date,customer_id,product_name;
```
---

**Bonus Query #2** Rank All The Things - Recreate the following table output using the available data


| customer_id | order_date               | product_name | price | member | ranking |
| ----------- | ------------------------ | ------------ | ----- | ------ | ------- |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |         |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |         |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      | 1       |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      | 2       |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |         |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |         |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |         |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      | 1       |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      | 2       |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |         |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |         |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |         |




``` sql
WITH status AS
(
SELECT 
  s.customer_id,
  s.order_date,
  m.product_name,
  m.price,
  CASE 
    WHEN order_date < join_date OR join_date is null
      THEN 'N'
    ELSE 'Y'
  END member
FROM dannys_diner.sales s
LEFT JOIN dannys_diner.members mem USING(customer_id)
LEFT JOIN dannys_diner.menu m USING (product_id)
ORDER BY customer_id, order_date
)

SELECT *,
 CASE
   WHEN member = 'N'
     THEN null
   ELSE DENSE_RANK() OVER (PARTITION BY customer_id, member ORDER BY order_date)							 
 END ranking
FROM status s;
```

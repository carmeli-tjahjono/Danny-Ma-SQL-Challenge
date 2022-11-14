Table 1: sales

The sales table captures all customer_id level purchases with an corresponding order_date and product_id information for when and what menu items were ordered.

![Sales Table](https://user-images.githubusercontent.com/115651033/201737201-041cb3e0-0bf6-4c79-a474-e90e686c189c.png)

Table 2: menu

The menu table maps the product_id to the actual product_name and price of each menu item.

![Product Table](https://user-images.githubusercontent.com/115651033/201737231-0934fd35-684c-4ad9-89a7-c2c32a94c2f2.png)

Table 3: members

The final members table captures the join_date when a customer_id joined the beta version of the Danny’s Diner loyalty program.

![Member Table](https://user-images.githubusercontent.com/115651033/201737247-584e0c9d-38ce-480f-a9a4-65fdda8bb6f5.png)


1. What is the total amount each customer spent at the restaurant?

SELECT customer_id, SUM(product_id*price) AS total_spent
FROM dannys_diner.sales
JOIN dannys_diner.menu
USING (product_id)
GROUP BY customer_id
ORDER BY customer_id

2. How many days has each customer visited the restaurant?

SELECT customer_id, COUNT(DISTINCT order_date) as total_days
FROM dannys_diner.sales
GROUP BY customer_id

3. What was the first item from the menu purchased by each customer?

SELECT customer_id, product_name, rank
FROM 
(
  SELECT customer_id, product_id, 
		DENSE_RANK () OVER (PARTITION BY customer_id ORDER BY order_date) AS rank
	FROM dannys_diner.sales
	GROUP BY customer_id, product_id, order_date
) AS rank_table
JOIN dannys_diner.menu
USING (product_id)
WHERE rank =1
ORDER BY customer_id

4. What is the most purchased item on the menu and how many times was it purchased by all customers?

SELECT product_name, COUNT (product_id)
FROM dannys_diner.sales
JOIN dannys_diner.menu
USING (product_id)
GROUP BY product_name
ORDER BY count DESC
LIMIT 1

5. Which item was the most popular for each customer?

WITH rank AS (
  SELECT customer_id, product_name, COUNT(product_id),
  		DENSE_RANK () OVER (PARTITION BY customer_id ORDER BY product_id DESC) as rank
  FROM dannys_diner.menu
  JOIN dannys_diner.sales
  USING (product_id)
  GROUP BY customer_id, product_id, product_name
  )

SELECT customer_id, product_name, count
  FROM rank
  WHERE rank = 1
  
6. Which item was purchased first by the customer after they became a member?

WITH rank AS
( SELECT customer_id, product_id, order_date, join_date,
		DENSE_RANK () OVER (PARTITION BY customer_id ORDER BY order_date) AS rank
	FROM dannys_diner.sales
 	JOIN dannys_diner.members
 	USING (customer_id)
	WHERE order_date >= join_date
 	GROUP BY customer_id, product_id, order_date, join_date
 )
    
--SELECT customer_id, product_name
--FROM rank
--JOIN dannys_diner.menu
--USING (product_id)
--WHERE rank =1

7. Which item was purchased just before the customer became a member?

WITH rank AS
( SELECT customer_id, product_id, order_date, join_date,
		DENSE_RANK () OVER (PARTITION BY customer_id ORDER BY order_date) AS rank
	FROM dannys_diner.sales
 	JOIN dannys_diner.members
	USING (customer_id)
	WHERE order_date < join_date
 	GROUP BY customer_id, product_id, order_date, join_date
 )
    
SELECT customer_id, product_name
FROM rank
JOIN dannys_diner.menu
USING (product_id)
WHERE rank =1
ORDER BY customer_id

8. What is the total items and amount spent for each member before they became a member?

WITH rank AS
( SELECT customer_id, product_id, order_date, join_date,
		DENSE_RANK () OVER (PARTITION BY customer_id ORDER BY order_date) AS rank
	FROM dannys_diner.sales
 	JOIN dannys_diner.members
	USING (customer_id)
	WHERE order_date < join_date
 	GROUP BY customer_id, product_id, order_date, join_date
 )
    
SELECT customer_id, COUNT(product_id) AS total_items, SUM(product_id*price) AS total_spent
FROM rank
JOIN dannys_diner.menu
USING (product_id)
WHERE rank =1
GROUP BY customer_id
ORDER BY customer_id

9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

WITH points AS
( SELECT product_id, product_name, price,
 	CASE WHEN product_id = 1 THEN price*20
 	ELSE price*10
  END AS points
FROM dannys_diner.menu
)

SELECT customer_id, SUM(points)
FROM points
JOIN dannys_diner.sales
USING (product_id)
GROUP BY customer_id

10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?


WITH  first_week AS
 (
   SELECT customer_id, join_date, join_date + INTERVAL '6 days' AS first_week
   FROM dannys_diner.members
  )
  
SELECT customer_id,  
	 SUM(
       CASE WHEN product_id = 1 THEN price*2*10
      	WHEN order_date BETWEEN join_date AND first_week THEN price*2*10
      ELSE price*10
      END) AS points
FROM dannys_diner.sales
JOIN first_week
USING(customer_id)
JOIN dannys_diner.menu
USING (product_id)
GROUP BY customer_id

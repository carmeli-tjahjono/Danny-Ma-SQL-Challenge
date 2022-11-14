# Tables
Table 1: sales

The sales table captures all customer_id level purchases with an corresponding order_date and product_id information for when and what menu items were ordered.

| customer_id | order_date               | product_id |
| ----------- | ------------------------ | ---------- |
| A           | 2021-01-01T00:00:00.000Z | 1          |
| A           | 2021-01-01T00:00:00.000Z | 2          |
| A           | 2021-01-07T00:00:00.000Z | 2          |
| A           | 2021-01-10T00:00:00.000Z | 3          |
| A           | 2021-01-11T00:00:00.000Z | 3          |
| A           | 2021-01-11T00:00:00.000Z | 3          |
| B           | 2021-01-01T00:00:00.000Z | 2          |
| B           | 2021-01-02T00:00:00.000Z | 2          |
| B           | 2021-01-04T00:00:00.000Z | 1          |
| B           | 2021-01-11T00:00:00.000Z | 1          |
| B           | 2021-01-16T00:00:00.000Z | 3          |
| B           | 2021-02-01T00:00:00.000Z | 3          |
| C           | 2021-01-01T00:00:00.000Z | 3          |
| C           | 2021-01-01T00:00:00.000Z | 3          |
| C           | 2021-01-07T00:00:00.000Z | 3          |


Table 2: menu

The menu table maps the product_id to the actual product_name and price of each menu item.

| product_id | product_name | price |
| ---------- | ------------ | ----- |
| 1          | sushi        | 10    |
| 2          | curry        | 15    |
| 3          | ramen        | 12    |



Table 3: members

The final members table captures the join_date when a customer_id joined the beta version of the Danny’s Diner loyalty program.

| customer_id | join_date                |
| ----------- | ------------------------ |
| A           | 2021-01-07T00:00:00.000Z |
| B           | 2021-01-09T00:00:00.000Z |


# Case Study Questions

1. What is the total amount each customer spent at the restaurant?

    SELECT customer_id, SUM(product_id*price) AS total_spent
    FROM dannys_diner.sales
    JOIN dannys_diner.menu
    USING (product_id)
    GROUP BY customer_id
    ORDER BY customer_id;

| customer_id | total_spent |
| ----------- | ----------- |
| A           | 178         |
| B           | 152         |
| C           | 108         |


2. How many days has each customer visited the restaurant?

    SELECT customer_id, COUNT(DISTINCT order_date) as total_days
    FROM dannys_diner.sales
    GROUP BY customer_id;

| customer_id | total_days |
| ----------- | ---------- |
| A           | 4          |
| B           | 6          |
| C           | 2          |



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
    ORDER BY customer_id;

| customer_id | product_name | rank |
| ----------- | ------------ | ---- |
| A           | sushi        | 1    |
| A           | curry        | 1    |
| B           | curry        | 1    |
| C           | ramen        | 1    |



4. What is the most purchased item on the menu and how many times was it purchased by all customers?

    SELECT product_name, COUNT (product_id)
    FROM dannys_diner.sales
    JOIN dannys_diner.menu
    USING (product_id)
    GROUP BY product_name
    ORDER BY count DESC
    LIMIT 1;

| product_name | count |
| ------------ | ----- |
| ramen        | 8     |


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
      WHERE rank = 1;

| customer_id | product_name | count |
| ----------- | ------------ | ----- |
| A           | ramen        | 3     |
| B           | ramen        | 2     |
| C           | ramen        | 3     |


  
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
        
    SELECT customer_id, product_name
    FROM rank
    JOIN dannys_diner.menu
    USING (product_id)
    WHERE rank =1;

| customer_id | product_name |
| ----------- | ------------ |
| B           | sushi        |
| A           | curry        |



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
    ORDER BY customer_id;

| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| A           | curry        |
| B           | curry        |


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
    ORDER BY customer_id;

| customer_id | total_items | total_spent |
| ----------- | ----------- | ----------- |
| A           | 2           | 40          |
| B           | 1           | 30          |


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
    GROUP BY customer_id;

| customer_id | sum |
| ----------- | --- |
| B           | 940 |
| C           | 360 |
| A           | 860 |


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
    GROUP BY customer_id;

| customer_id | points |
| ----------- | ------ |
| A           | 1370   |
| B           | 940    |



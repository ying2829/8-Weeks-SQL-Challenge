#  Case Study #4: Data Bank

Welcome to the Week-4---Data Bank

### A. Customer Nodes Exploration

1. How many unique nodes are there on the Data Bank system?

```Mysql
SELECT COUNT(DISTINCT node_id) AS number_of_node
FROM customer_nodes
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/3c705edb-97b7-4f66-979f-a882e83bd9e4)


2. What is the number of nodes per region?

```Mysql
SELECT region_name, COUNT(DISTINCT node_id) AS number_of_node
FROM customer_nodes
JOIN regions USING (region_id)
GROUP BY region_name;
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/65cd0fc2-aff9-4619-9677-8c0986fe4440)


3. How many customers are allocated to each region?

```Mysql
SELECT region_name, COUNT(DISTINCT customer_id) AS number_of_customer
FROM customer_nodes
JOIN regions USING (region_id)
GROUP BY region_name;
```

![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/46c524d1-4415-4f9d-8f0d-170161b80889)

4. How many days on average are customers reallocated to a different node?

First of all, we should extract the date the customer changed the node, so I used the LAG function to pick up the last date the customer changed the node. And, I created the CTE to reduce the view creating. Plus, in this part, I don't recommend creating a view cause this is not a basic view and is not beneficial for the following analysis.

```Mysql
SELECT customer_id, node_id, start_date,end_date,
LAG(start_date,1) OVER (PARTITION BY customer_id ORDER BY start_date,node_id) AS last_date
FROM customer_nodes;
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/35a3bc02-ced6-4ac6-a849-c684b693abad)

```Mysql
WITH change_date AS (SELECT customer_id, node_id, start_date,end_date,
LAG(start_date,1) OVER (PARTITION BY customer_id ORDER BY start_date,node_id) AS last_date
FROM customer_nodes)
SELECT AVG( DATEDIFF(start_date,last_date)) AS avg_change_date
FROM change_date
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/0223dfa8-e151-4afa-975b-1d9307b4fa30)


5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

For the median:

```Mysql
WITH change_date AS (SELECT customer_id, node_id, start_date,end_date,region_name,
LAG(start_date,1) OVER (PARTITION BY customer_id ORDER BY start_date,node_id) AS last_date
FROM customer_nodes
JOIN regions USING (region_id)),
region_group AS (SELECT region_name, DATEDIFF(start_date,last_date) AS date_diff
FROM change_date
GROUP BY region_name,DATEDIFF(start_date,last_date)
HAVING date_diff IS NOT NULL
ORDER BY region_name,date_diff),
rank_date AS (SELECT region_name, date_diff, ROW_NUMBER() OVER(PARTITION BY region_name ORDER BY date_diff) AS date_order, 
COUNT(*) OVER( PARTITION BY region_name) AS cnt
FROM region_group)
SELECT region_name, date_diff
FROM rank_date
WHERE date_order IN ((cnt+1)/2, (cnt+2)/2, CEIL(0.8 * cnt), CEIL(0.95 * cnt))
ORDER BY region_name
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/a7ab6de0-edd5-4247-9439-8f2c96bf1d19)

### B. Customer Transactions

1. What is the unique count and total amount for each transaction type?

```Mysql
SELECT txn_type,COUNT(txn_type) AS total_txn_type, SUM(txn_amount) AS total_amount
FROM customer_transactions
GROUP BY txn_type
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/504540b0-770c-439c-a87b-db0d675aa711)

2. What is the average total historical deposit counts and amounts for all customers?

```Mysql
SELECT txn_type,
COUNT(txn_type)/COUNT(DISTINCT(customer_id)) AS avg_txn_type, SUM(txn_amount)/ COUNT(DISTINCT(customer_id)) AS avg_amount
FROM customer_transactions
GROUP BY txn_type
HAVING txn_type="deposit"
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/41729436-7c61-47de-bff5-ea16976d6f31)

3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
 
In this part, first of all, we need to tidy up the data count of the deposit, withdrawal, and purchase made by each customer. And then, we can filter the condition in the question which is the deposit should be more than 1 and either 1 purchase or 1 withdrawal. In the end, count the distinct customer and group by each month. Please see below:

```Mysql
WITH cte AS (SELECT customer_i,MONTH(txn_date) AS month_id,MONTHNAME(txn_date) AS month_name,
COUNT(CASE WHEN txn_type = 'deposit' THEN 1 END) AS deposit_count,
COUNT(CASE WHEN txn_type = 'purchase' THEN 1 END) AS purchase_count,
COUNT(CASE WHEN txn_type = 'withdrawal' THEN 1 END) AS withdrawal_count
FROM customer_transactions
GROUP BY customer_id, MONTH(txn_date), MONTHNAME(txn_date)
HAVING deposit_count>1  AND (purchase_count > 0 OR withdrawal_count > 0)
ORDER BY customer_id)
SELECT month_id,month_name, COUNT(DISTINCT(customer_id)) AS total
FROM cte
GROUP BY month_id,month_name
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/96b2d15b-c942-4139-93fb-1bfea67aba02)

4. What is the closing balance for each customer at the end of the month?

```Mysql
WITH cte AS (SELECT customer_id,MONTH(txn_date) AS month_id,MONTHNAME(txn_date) AS month_name ,txn_type,
CASE WHEN txn_type="purchase" THEN 0-txn_amount
WHEN txn_type="withdrawal" THEN 0-txn_amount
ELSE txn_amount
END AS amount
FROM customer_transactions
ORDER BY customer_id,MONTH(txn_date)),
cte1 AS (SELECT customer_id,month_id,month_name,
SUM(CASE WHEN txn_type = 'deposit' THEN amount ELSE 0 END) AS deposit,
SUM(CASE WHEN txn_type = 'purchase' THEN amount ELSE 0 END) AS purchase,
SUM(CASE WHEN txn_type = 'withdrawal' THEN amount ELSE 0 END) AS withdrawal
FROM cte
GROUP BY customer_id,month_id,month_name
ORDER BY customer_id,MONTH(txn_date))
SELECT customer_id,month_id,month_name,deposit+purchase+withdrawal AS total
FROM cte1
GROUP BY customer_id,month_id,month_name
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/e6821b87-a69a-4f01-9cae-d4f188b68f5d)


5. What is the percentage of customers who increase their closing balance by more than 5%?
This case is an extension of question 4. So, from the last question, we have a table about the end balance of each month for each customer. Therefore, we only need to use the LEAD function to fetch the amount of the following balance. Then, for the question which is to find the percentage of the customer who increases their closing balance by 5%, you will know from the table there something we need to be very careful which is the positive and negative values. So, we need to use absolute value to calculate and filter the value is lower than ±5%. In the end, we can calculate the percentage of the customer who fit to the condition.
```Mysql
USE data_bank;
WITH cte AS (SELECT customer_id,MONTH(txn_date) AS month_id,MONTHNAME(txn_date) AS month_name ,txn_type,
CASE WHEN txn_type="purchase" THEN 0-txn_amount
WHEN txn_type="withdrawal" THEN 0-txn_amount
ELSE txn_amount
END AS amount
FROM customer_transactions
ORDER BY customer_id,MONTH(txn_date)),
cte1 AS (SELECT customer_id,month_id,month_name,
SUM(CASE WHEN txn_type = 'deposit' THEN amount ELSE 0 END) AS deposit,
SUM(CASE WHEN txn_type = 'purchase' THEN amount ELSE 0 END) AS purchase,
SUM(CASE WHEN txn_type = 'withdrawal' THEN amount ELSE 0 END) AS withdrawal
FROM cte
GROUP BY customer_id,month_id,month_name
ORDER BY customer_id,MONTH(txn_date)),
cte2 AS (SELECT customer_id,month_id,month_name,deposit+purchase+withdrawal AS total
FROM cte1
GROUP BY customer_id,month_id,month_name),
cte3 AS (SELECT customer_id,month_id,month_name,total,
LEAD(total) OVER( PARTITION BY customer_id ORDER BY month_id) AS following_balance
FROM cte2),
cte4 AS (SELECT customer_id,month_id,month_name,
ROUND( (ABS(total)-ABS(following_balance))/ABS(total)) AS increase_percentage
FROM cte3),
cte5 AS (SELECT customer_id,increase_percentage
FROM cte4
WHERE increase_percentage IS NOT NULL AND (increase_percentage >= 5 OR increase_percentage <= -5))
SELECT CONCAT(COUNT(DISTINCT(customer_id))/500*100,"%") AS percentage
FROM cte5
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/3fb8dba4-f6c9-4e64-a3df-ddf178fff269)


### C. Data Allocation Challenge

To test out a few different hypotheses - the Data Bank team wants to run an experiment where different groups of customers would be allocated data using 3 different options:


* Option 3: data is updated real-time
For this multi-part challenge question - you have been requested to generate the following data elements to help the Data Bank team estimate how much data will need to be provisioned for each option:

* running customer balance column that includes the impact each transaction
```Mysql
WITH cte AS (SELECT customer_id,txn_date,txn_type,
CASE WHEN txn_type="deposit" THEN txn_amount
ELSE -txn_amount
END AS balance
FROM customer_transactions
ORDER BY customer_id)
SELECT customer_id,txn_date,txn_type, balance,
SUM(balance) OVER (PARTITION BY customer_id ORDER BY customer_id ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS rest_balance
FROM cte
GROUP BY customer_id, txn_date,txn_type, balance
ORDER BY customer_id;
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/da5f3e4d-d7fb-4313-943e-afc4e3d49e43)

* customer balance at the end of each month

This is identical to question 4 in part B.
* minimum, average and maximum values of the running balance for each customer
```Mysql
WITH cte AS (SELECT customer_id,txn_date,txn_type,
CASE WHEN txn_type="deposit" THEN txn_amount
ELSE -txn_amount
END AS balance
FROM customer_transactions
ORDER BY customer_id),
cte1 AS (SELECT customer_id,txn_date,txn_type, balance,
SUM(balance) OVER (PARTITION BY customer_id ORDER BY customer_id ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS rest_balance
FROM cte
GROUP BY customer_id, txn_date,txn_type, balance
ORDER BY customer_id)
SELECT customer_id, 
MIN(rest_balance)  AS minimum_balance,
MAX(rest_balance)  AS max_balance,
AVG(rest_balance) AS avg_balance
FROM cte1
GROUP BY customer_id
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/1686928c-c1be-42c2-ad6d-bc072bdfec07)

After generating the data elements, now we can have a look at each option:

* **Option 1: data is allocated based off the amount of money at the end of the previous month**
```Mysql
WITH cte AS (SELECT customer_id,MONTH(txn_date) AS month_id,MONTHNAME(txn_date) AS month_name ,txn_type,
CASE WHEN txn_type="purchase" THEN 0-txn_amount
WHEN txn_type="withdrawal" THEN 0-txn_amount
ELSE txn_amount
END AS amount
FROM customer_transactions
ORDER BY customer_id,MONTH(txn_date)),
cte1 AS (SELECT customer_id,month_id,month_name,
SUM(CASE WHEN txn_type = 'deposit' THEN amount ELSE 0 END) AS deposit,
SUM(CASE WHEN txn_type = 'purchase' THEN amount ELSE 0 END) AS purchase,
SUM(CASE WHEN txn_type = 'withdrawal' THEN amount ELSE 0 END) AS withdrawal
FROM cte
GROUP BY customer_id,month_id,month_name
ORDER BY customer_id,MONTH(txn_date)),
cte2 AS (SELECT customer_id,month_id,month_name,deposit+purchase+withdrawal AS total
FROM cte1
GROUP BY customer_id,month_id,month_name),
cte3 AS (SELECT customer_id,month_id,month_name,total, LAG(total) OVER(PARTITION BY customer_id ORDER BY customer_id) AS last_month_balance
FROM cte2)
SELECT month_id,month_name,SUM(last_month_balance) AS total_allocation
FROM cte3
GROUP BY month_id,month_name
ORDER BY month_id
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/0cfa88f6-cd22-4b19-9334-8799a60619b7)

From the table, we can see that all the allocation is based on the previous month, so there is no data in Jan. Also, the reason for the negative value is that some accounts of the customers hold a negative balance. However, it is not appropriate for customers to receive negative storage. Therefore, I made an assumption that the negative balance was a debit, I converted it into a positive, and then I could have a full page of the accurate total allocation storage.

```MySQL
WITH cte AS (SELECT customer_id,MONTH(txn_date) AS month_id,MONTHNAME(txn_date) AS month_name ,txn_type,
CASE WHEN txn_type="purchase" THEN 0-txn_amount
WHEN txn_type="withdrawal" THEN 0-txn_amount
ELSE txn_amount
END AS amount
FROM customer_transactions
ORDER BY customer_id,MONTH(txn_date)),
cte1 AS (SELECT customer_id,month_id,month_name,
SUM(CASE WHEN txn_type = 'deposit' THEN amount ELSE 0 END) AS deposit,
SUM(CASE WHEN txn_type = 'purchase' THEN amount ELSE 0 END) AS purchase,
SUM(CASE WHEN txn_type = 'withdrawal' THEN amount ELSE 0 END) AS withdrawal
FROM cte
GROUP BY customer_id,month_id,month_name
ORDER BY customer_id,MONTH(txn_date)),
cte2 AS (SELECT customer_id,month_id,month_name,deposit+purchase+withdrawal AS total
FROM cte1
GROUP BY customer_id,month_id,month_name),
cte3 AS (SELECT customer_id,month_id,month_name,total, LAG(total) OVER(PARTITION BY customer_id ORDER BY customer_id) AS last_month_balance
FROM cte2)
SELECT month_id,month_name, 
SUM(ABS(last_month_balance)) AS last_month_balance
FROM cte3
GROUP BY month_id,month_name
HAVING last_month_balance IS NOT NULL
ORDER BY month_id
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/61e885b2-f770-4552-b2b9-944b1ecc8340)

In this case, I put the absolute value to make sure all of them are positive. 

_Insight_
1. From the observation period, it indicated that the allocation data is flexible which means that Data usage or requirements may fluctuate at different times.
2. The amount in March is a peak compared to the rest months. And, compared to February, it shows a 120% increase in data storage needs or a higher demand for data resources.
3. In April, it indicated that there was a nearly 71% drop compared to last month.


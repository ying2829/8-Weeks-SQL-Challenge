# 🥑 Case Study #3: Foodie-Fi
### A. Customer Journey

Q1: Based off the 8 sample customers provided in the sample from the subscriptions table, write a brief description about each customer’s onboarding journey.Try to keep it as short as possible - you may also want to run some sort of join to make your explanations a bit easier!

Create Raw data to analyse the following questions:
1. customer changing plan journey, how many routes happened within these 8 customers?
2. How long does each customer take to swift to another plan?
3. How much does we earn from each customer?

**Ans 1/3:**
```mysql
CREATE OR REPLACE VIEW customer_journey AS
SELECT customer_id,GROUP_CONCAT(plan_name SEPARATOR' - ') AS customer_map, 
GROUP_CONCAT(start_date SEPARATOR',') AS time_diff,
SUM(price) AS total_paid
FROM subscriptions 
JOIN plans USING (plan_id)
GROUP BY customer_id
HAVING customer_id<=8;
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/ecd2c8e2-29eb-4b43-9884-f7bc6640d30d)


Then we can find out only four paths have been conducted within these 8 customers. Which by use the following:

```mysql
SELECT customer_map,COUNT(customer_map)
FROM customer_journey
GROUP BY customer_map
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/dc588336-e277-4214-8698-01cc543bd172)


**Ans 2:**
By creating a view named as time change, we can see how much time each customer takes to change a plan. 

```mysql
CREATE OR REPLACE VIEW plan_time AS
SELECT customer_id,
SUBSTRING(time_diff,1,10)AS first_register,
SUBSTRING(time_diff,12,10) AS second_change,
SUBSTRING(time_diff,23,10) AS third_change
FROM customer_journey;
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/e1e1bafd-7374-4968-8af3-b7ac00a92a14)

And then, I would suggest to do some calculation:
```mysql
SELECT customer_id,datediff(second_change,first_register) AS transform_1,
DATEDIFF(third_change,second_change) AS transform_2
FROM plan_time;
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/0d859e1d-4635-415f-afc2-883da09f2bb5)

**Summary:**
Four paths have been conducted within these 8 customers. And, every customer continued with another plan after 7 days of free trial. 25% of customers cancelled the plan. And, 50% of customers changed plans twice.

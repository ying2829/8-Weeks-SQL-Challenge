# 🥑 Case Study #3: Foodie-Fi
Welcome to the Week-3---Foodie-Fi 

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

### B. Data Analysis Questions

Q1. How many customers has Foodie-Fi ever had?

```mysql
SELECT COUNT(DISTINCT(customer_id))
FROM subscriptions
```
![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/29689c74-7e3a-4f0d-8738-3518ea499287)

Q2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value

```mysql
USE foodie_fi;
SELECT MONTH(start_date) AS month_trial,plan_name,COUNT(customer_id) AS number_customer
FROM subscriptions
JOIN plans USING(plan_id)
GROUP BY month_trial,plan_name 
HAVING plan_name="trial"
ORDER BY month_trial
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/cb293502-2acc-41b9-bd4f-c1524dcd477b)

Q3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name

```mysql
SELECT plan_name,COUNT(plan_name) AS number,
IF (start_date>"2020-12-30", "now","archieve") AS category
FROM subscriptions
JOIN plans USING(plan_id)
GROUP BY plan_name, category
HAVING category="now"
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/cbea49b0-7269-4cb6-84b8-aa5be1d34d57)

Q4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

For this one, we all know that all the customers need to start with a trial plan, so I use the count of the trial to know the total mount of the customer is 1000.

```mysql
SELECT plan_name, COUNT(*) AS total,
CONCAT(ROUND(COUNT(*)/1000*100,1),"%") AS percentage
FROM subscriptions
JOIN plans USING(plan_id)
GROUP BY plan_name;
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/15b7af01-645b-4958-a4e1-77bb263890d2)

Q5/6. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?/ What is the number and percentage of customer plans after their initial free trial?

Again, we all know the total amount of the customer is 1000, so it would be easier to combine the code from questions 1 and 3 from the A part. And then, deleted the having clause.

```mysql
CREATE OR REPLACE VIEW total_customer AS
SELECT customer_id,GROUP_CONCAT(plan_name SEPARATOR' - ') AS customer_map, 
GROUP_CONCAT(start_date SEPARATOR',') AS time_diff,
SUM(price) AS total_paid
FROM subscriptions 
JOIN plans USING (plan_id)
GROUP BY customer_id;
SELECT customer_map,CONCAT(ROUND(COUNT(customer_map)/1000*100,1),"%") as percentage
FROM total_customer 
GROUP BY customer_map
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/d3e528ac-1c00-43bc-8636-ea0c66d08f27)

Q7/8. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?/ How many customers have upgraded to an annual plan in 2020?

```mysql
SELECT plan_name,COUNT(plan_name) AS total_number, CONCAT(ROUND(COUNT(plan_name)/1000*100,1),"%") AS percentage,
IF(start_date<="2020-12-31","Archive","Now") AS category
FROM subscriptions
JOIN plans USING (plan_id)
GROUP BY plan_name,category
HAVING category="Archive"
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/0747e896-6b98-421b-8192-cc63a6b9143d)

Q9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?

```mysql
CREATE OR REPLACE VIEW trial_plan AS
SELECT customer_id,start_date AS trial_date
FROM subscriptions
WHERE plan_id=0;
CREATE OR REPLACE VIEW pro_plan AS
SELECT customer_id,start_date AS pro_date
FROM subscriptions
WHERE plan_id=3;
SELECT ROUND(AVG(DATEDIFF(pro_date,trial_date)),1) AS upgrade
FROM trial_plan t
JOIN pro_plan p USING(customer_id)
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/0cc5d206-089d-4351-ad50-f46b805e9dfd)


Q10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)

```mysql
USE foodie_fi;
SELECT 
CASE WHEN DATEDIFF(pro_date,trial_date)<=30 THEN "0-30 days"
WHEN  DATEDIFF(pro_date,trial_date)>30 AND DATEDIFF(pro_date,trial_date)<=60 THEN "30-60 days"
WHEN DATEDIFF(pro_date,trial_date)>60 AND DATEDIFF(pro_date,trial_date)<=90 THEN "60-90 days"
WHEN DATEDIFF(pro_date,trial_date)>90 AND DATEDIFF(pro_date,trial_date)<=120 THEN "90-120 days"
WHEN DATEDIFF(pro_date,trial_date)>120 AND DATEDIFF(pro_date,trial_date)<=150 THEN "120-150 days"
WHEN DATEDIFF(pro_date,trial_date)>150 AND DATEDIFF(pro_date,trial_date)<=180 THEN "150-180 days"
WHEN DATEDIFF(pro_date,trial_date)>180 AND DATEDIFF(pro_date,trial_date)<=210 THEN "180-210 days"
WHEN DATEDIFF(pro_date,trial_date)>210 AND DATEDIFF(pro_date,trial_date)<=240 THEN "210-240 days"
WHEN DATEDIFF(pro_date,trial_date)>240 AND DATEDIFF(pro_date,trial_date)<=270 THEN "240-270 days"
WHEN DATEDIFF(pro_date,trial_date)>270 AND DATEDIFF(pro_date,trial_date)<=300 THEN "270-300 days"
WHEN DATEDIFF(pro_date,trial_date)>300 AND DATEDIFF(pro_date,trial_date)<=330 THEN "300-330 days"
ELSE "330-360 days"
END AS bucket,
COUNT(*) AS customer_count
FROM trial_plan t
JOIN pro_plan p USING(customer_id)
GROUP BY bucket
ORDER BY bucket
```
![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/3eac5304-4c39-4516-aa7d-95895b12d8ba)

Q11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?

```mysql
CREATE OR REPLACE VIEW basic_plan AS
SELECT customer_id,start_date AS basic_date
FROM subscriptions
WHERE plan_id=1;
CREATE OR REPLACE VIEW prmonthly_plan AS
SELECT customer_id,start_date AS prmonthly_date
FROM subscriptions
WHERE plan_id=2;
SELECT
COUNT(IF(basic_date>prmonthly_date,1,Null)) AS downgrade
FROM basic_plan 
JOIN prmonthly_plan  USING(customer_id)
WHERE basic_date<"2021-01-01" AND prmonthly_date<"2021-01-01"
```

There is no customer downgraded from pro monthly to basic monthly in 2020.

### C. Challenge Payment Question
The Foodie-Fi team wants you to create a new payments table for the year 2020 that includes amounts paid by each customer in the subscriptions table with the following requirements:

* monthly payments always occur on the same day of month as the original start_date of any monthly paid plan
* upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately
* upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period
* once a customer churns they will no longer make payments

Ans:
So, if we need to create a table for payment for 2020, I will suggest that we need to fetch the data with the following requests which is:

1. All the data should within 2020
2. Eliminate the "trial" and "churn" because there is no payment will be made
3. Fetch the date when the customer changed the plan

```mysql
USE foodie_fi;
SELECT customer_id, plan_id, plan_name, start_date, price,
     COALESCE( LEAD(start_date, 1) OVER(PARTITION BY customer_id ORDER BY start_date, plan_id),"2020-12-31" )AS cutoff_date
FROM subscriptions
JOIN plans
USING (plan_id)
WHERE start_date BETWEEN "2020-01-01" AND "2020-12-31"
 AND plan_name NOT IN("trial", "churn")
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/9de3d0d5-e177-4866-86a9-fa2fe719eec1)


And then, according to the plan name, we all know the customers who stay in xx_monthly will make a payment monthly until the end of 2020. However, those who subscribe for pro_annual only pay once for the whole year. So, to track the payment, we need to create a recursive table for 2020_payment table.

```mysql
WITH RECURSIVE cte AS (
 SELECT customer_id, plan_id, plan_name, start_date, price,
  COALESCE( LEAD(start_date, 1) OVER(PARTITION BY customer_id ORDER BY start_date, plan_id),"2020-12-31" )AS cutoff_date
FROM subscriptions
JOIN plans
USING (plan_id)
WHERE start_date BETWEEN "2020-01-01" AND "2020-12-31"
 AND plan_name NOT IN("trial", "churn")
 UNION ALL
 SELECT customer_id, plan_id, plan_name, 
    DATE((start_date + INTERVAL 1 MONTH)) AS start_date, price, cutoff_date
    FROM cte
    WHERE cutoff_date > DATE((start_date + INTERVAL 1 MONTH))
   AND plan_name <> "pro annual")
SELECT * FROM cte
ORDER BY customer_id, start_date;
```

![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/dfb531a9-9b99-4b47-9d42-c42459a9094c)


However, this is not the end. In order to fulfill the last condition which is upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately

```mysql
USE foodie_fi;
CREATE TABLE payment_2020 AS
WITH RECURSIVE cte AS (
 SELECT customer_id, plan_id, plan_name, start_date, price,
  COALESCE( LEAD(start_date, 1) OVER(PARTITION BY customer_id ORDER BY start_date, plan_id),"2020-12-31" )AS cutoff_date
FROM subscriptions
JOIN plans
USING (plan_id)
WHERE start_date BETWEEN "2020-01-01" AND "2020-12-31"
 AND plan_name NOT IN("trial", "churn")
 UNION ALL
 SELECT customer_id, plan_id, plan_name, 
    DATE((start_date + INTERVAL 1 MONTH)) AS start_date, price, cutoff_date
    FROM cte
    WHERE cutoff_date > DATE((start_date + INTERVAL 1 MONTH))
   AND plan_name <> "pro annual"),
cte1 AS (
 SELECT *, 
   LAG(plan_id, 1) OVER(PARTITION BY customer_id ORDER BY start_date) 
    AS last_payment_plan,
   LAG(price, 1) OVER(PARTITION BY customer_id ORDER BY start_date) 
    AS last_amount_paid,
   RANK() OVER(PARTITION BY customer_id ORDER BY start_date) AS payment_order
 FROM cte
 ORDER BY customer_id, start_date
)
SELECT customer_id, plan_id, plan_name, start_date AS payment_date, 
 (CASE 
   WHEN plan_id IN (2, 3) AND last_payment_plan = 1 
    THEN price - last_amount_paid
   ELSE price
 END) AS price, payment_order
FROM cte1;
```
In this session, I used the obvious part to show what I mean. So, for customer_id =16, he changed his plan from basic monthly to pro annually, which means the price paid for pro annually should be deducted from what he paid for the basic monthly.
![image](https://github.com/ying2829/Weeek-3---Foodie-Fi/assets/162821565/f75eeb3f-186a-4524-af3a-9823ea472af1)

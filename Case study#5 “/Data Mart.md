#  Case Study #5: Data Mart

Welcome to the Week-5---Data Mart

### A. Data Cleansing Steps
In a single query, perform the following operations and generate a new table in the data_mart schema named clean_weekly_sales:

Convert the week_date to a DATE format

Add a week_number as the second column for each week_date value, for example any value from the 1st of January to 7th of January will be 1, 8th to 14th will be 2 etc

Add a month_number with the calendar month for each week_date value as the 3rd column

Add a calendar_year column as the 4th column containing either 2018, 2019 or 2020 values

Add a new column called age_band after the original segment column using the following mapping on the number inside the segment value
```Mysql
WITH cte AS (SELECT STR_TO_DATE(week_date,'%d/%m/%y') AS week_date,region,platform,segment,customer_type,transactions,sales
FROM weekly_sales)
SELECT week_date,WEEK(week_date) AS week_number, MONTH(week_date) AS month_number, YEAR(week_date) AS calender_year,
region,platform,segment,
CASE WHEN segment REGEXP 1 THEN 'Young Adults'
WHEN segment REGEXP 2 THEN 'Middle Aged'
WHEN segment REGEXP '3|4' THEN 'Retirees'
ELSE 'unknown'
END AS age_band,
CASE WHEN segment REGEXP 'C' THEN 'Couples'
WHEN segment REGEXP 'F' THEN 'Families'
ELSE 'unknown'
END AS 'demographic',
customer_type,transactions,sales,
ROUND(sales/transactions,2) AS avg_transaction 
FROM cte
```

![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/3ef1f26c-3831-4660-8ca3-c2c669f40f30)


### B. Data Exploration

1. What day of the week is used for each week_date value?

```Mysql
USE data_mart;
WITH cte AS (SELECT week_number,DAYNAME(week_date) AS weekday
FROM clean_date
ORDER BY week_number)
SELECT weekday,COUNT(DISTINCT(week_number))
FROM cte
GROUP BY weekday
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/8a836eab-de99-4b6c-ba3d-ab725c274fc0)


2. What range of week numbers are missing from the dataset?

First of all, I want to know how many years the data contain, so I calculate it like below:
```Mysql
SELECT calender_year, COUNT(DISTINCT(calender_year))
FROM clean_date
GROUP BY calender_year
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/07bf1f24-1c02-482d-ac5d-658de9533d26)

Then I will create a temporary table as a filter that can help me understand the missing data.

```Mysql
CREATE TEMPORARY TABLE IF NOT EXISTS filter_weeknum
(calender_year INT NOT NULL, week_number INT);
WITH RECURSIVE cte AS (
    SELECT 2018 AS calendar_year, 1 AS week_number
    UNION ALL
    SELECT 
        CASE WHEN week_number < 52 THEN calendar_year ELSE calendar_year + 1 END,
        CASE WHEN week_number < 52 THEN week_number + 1 ELSE 1 END
    FROM cte
    WHERE week_number < 52 OR (week_number = 52 AND calendar_year < 2018 + 1) OR (week_number = 52 AND calendar_year < 2018 + 2)
)
SELECT *
FROM cte
LIMIT 156
```
![image](https://github.com/ying2829/8-Weeks-SQL-Challenge/assets/162821565/9ff0318f-5f84-4158-8a1f-762d3fcba004)

Then, I used JOIN to filter the current data and fetch the data range it missed.


3. How many total transactions were there for each year in the dataset?

```MySQL
USE data_mart;
SELECT calender_year, COUNT(transactions) AS total_transactions
FROM clean_date
GROUP BY calender_year
ORDER BY calender_year
```

![image](https://github.com/user-attachments/assets/31beeb46-713d-4cab-b4f2-672165982eb0)

4. What is the total sales for each region for each month?

For this question, I just realized that I accidentally set the sales into the Integer, so I need to change it to the decimal.Then, it would be really easy to calculate.

```MySQL
USE data_mart;
SELECT  region, month_number,SUM(sales) AS total_sales
FROM clean_date
GROUP BY region, month_number
ORDER BY region, month_number
```
![image](https://github.com/user-attachments/assets/8618392b-2ccf-40f2-b191-174fbcce0f5c)


5. What is the total count of transactions for each platform?

```MySQL
USE data_mart;
SELECT  platform, COUNT(transactions)AS total_transactions
FROM clean_date
GROUP BY platform
```
![image](https://github.com/user-attachments/assets/03849090-9a08-48bd-993c-1adb54c191ea)

6. What is the percentage of sales for Retail vs Shopify for each month?

```MySQL
WITH cte AS (
SELECT  platform, month_number,SUM(sales) AS total_sales, (SELECT SUM(sales)
FROM clean_date) AS total_sales_overall
FROM clean_date 
GROUP BY platform,month_number)
SELECT platform, month_number,CONCAT(total_sales/total_sales_overall*100,"%") AS percentage_of_sales
FROM cte
GROUP BY platform,month_number
ORDER BY platform,month_number
```
![image](https://github.com/user-attachments/assets/cd0018a8-dd1d-4618-bba8-39017fb43e8f)

7. What is the percentage of sales by demographic for each year in the dataset?

```MySQL
WITH cte AS (
SELECT  demographic, calender_year,SUM(sales) AS total_sales, (SELECT SUM(sales)
FROM clean_date) AS total_sales_overall
FROM clean_date 
GROUP BY demographic,calender_year)
SELECT demographic,calender_year,CONCAT(total_sales/total_sales_overall*100,"%") AS percentage_of_sales
FROM cte
GROUP BY demographic,calender_year
ORDER BY demographic,calender_year
```
![image](https://github.com/user-attachments/assets/066359b2-4576-4818-b4bb-40300cabcaa6)

8. Which age_band and demographic values contribute the most to Retail sales?

To answer this question, there is one thing worth mentioning. If we used the regular calculation method, the result should be unknown. But, I excluded this result because I don't think this can help the business. However, I will still provide this additional information when I presented the result.
```MySQL
SELECT platform,age_band,demographic, SUM(sales) AS total_sales
FROM clean_date
GROUP BY platform,age_band,demographic
HAVING platform="Retail" AND age_band != "unknown"
ORDER BY SUM(sales) DESC
LIMIT 1
```
![image](https://github.com/user-attachments/assets/265dde43-981b-49f9-b7b0-110cf8e684a5)

9. Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?

Unfortunately, we can't. It is simply to think mathematically. If we want to know the transaction size by platform and year, we should sum up the transaction and sales, and then use total sales divided by the total transaction. The result of avg_transaction is from each sale in each row divided by each transaction in each row. Therefore, if we use the average function for the avg_transaction, the numerator and denominator will be different. So, I suggested calculating the following:

```MySQL
WITH cte AS (SELECT calender_year,platform,
SUM(transactions) AS total_transaction,SUM(sales) AS total_sales
FROM clean_date
GROUP BY calender_year,platform)
SELECT calender_year,platform,total_sales/total_transaction AS transaction_size
FROM cte
GROUP BY calender_year,platform
ORDER BY calender_year ASC
```
![image](https://github.com/user-attachments/assets/c17ac59c-8777-4af0-afde-b4491955456c)

### C. Before & After Analysis

This technique is usually used when we inspect an important event and want to inspect the impact before and after a certain point in time.

Taking the week_date value of 2020-06-15 as the baseline week where the Data Mart sustainable packaging changes came into effect.

We would include all week_date values for 2020-06-15 as the start of the period after the change and the previous week_date values would be before

Using this analysis approach - answer the following questions:

* What is the total sales for the 4 weeks before and after 2020-06-15? What is the growth or reduction rate in actual values and percentage of sales?

First, we should know which week is 2020-06-15, and then we can categorize the data into before and after by definition. Once this is done, we can use the sum function to calculate. We can also use the result of the calculation to calculate the growth.

```MySQL
WITH cte1 AS (SELECT week_date,week_number,calender_year,
CASE WHEN week_date<"2020-06-15" THEN "before"
WHEN week_date="2020-06-15" THEN "baseline"
ELSE "after" 
END AS category, SUM(sales) AS total_sales
FROM clean_date
GROUP BY week_date,week_number,calender_year,
CASE WHEN week_date<"2020-06-15" THEN "before"
WHEN week_date="2020-06-15" THEN "baseline"
ELSE "after" 
END
HAVING calender_year="2020" AND week_number IN (20,21,22,23,25,26,27,28)),
cte2 AS (SELECT SUM(CASE WHEN category="before" THEN total_sales END) AS before_sales,
SUM(CASE WHEN category="after" THEN total_sales END) AS after_sales
FROM cte1)
SELECT after_sales-before_sales AS sale_variance,
ROUND((after_sales-before_sales)/before_sales*100,2) AS percentage_variance
FROM cte2
```
![image](https://github.com/user-attachments/assets/a042c12f-03fb-41a7-861b-9ad5b8dde0d3)




What about the entire 12 weeks before and after?

```MySQL
WITH cte1 AS (SELECT week_date,week_number,calender_year,
CASE WHEN week_date<"2020-06-15" THEN "before"
WHEN week_date="2020-06-15" THEN "baseline"
ELSE "after" 
END AS category, SUM(sales) AS total_sales
FROM clean_date
GROUP BY week_date,week_number,calender_year,
CASE WHEN week_date<"2020-06-15" THEN "before"
WHEN week_date="2020-06-15" THEN "baseline"
ELSE "after" 
END
HAVING calender_year="2020"),
cte2 AS (SELECT SUM(CASE WHEN category="before" THEN total_sales END) AS before_sales,
SUM(CASE WHEN category="after" THEN total_sales END) AS after_sales
FROM cte1)
SELECT after_sales-before_sales AS sale_variance,
ROUND((after_sales-before_sales)/before_sales*100,2) AS percentage_variance
FROM cte2
```
![image](https://github.com/user-attachments/assets/7e046d1c-c8c6-4998-8045-18d09af07239)

How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?

First, we need to clarify and define the timeframe to answer this question. 

* 4 weeks before and after period in 2018, 2019, and 2020.
```MySQL
WITH cte1 AS (SELECT week_number,calender_year, SUM(sales) AS total_sales
FROM clean_date
GROUP BY week_number,calender_year
HAVING week_number IN (20,21,22,23,25,26,27,28)),
cte2 AS (SELECT calender_year,
SUM(CASE WHEN week_number IN (20,21,22,23) THEN total_sales END) AS before_total_sales,
SUM(CASE WHEN week_number IN(25,26,27,28) THEN total_sales END) AS after_total_sales
FROM cte1
GROUP BY calender_year)
SELECT calender_year,after_total_sales-before_total_sales AS sale_variance,
ROUND((after_total_sales-before_total_sales)/before_total_sales*100,2) AS percentage_variance
FROM cte2
ORDER BY calender_year
```
![image](https://github.com/user-attachments/assets/e348aeab-bca9-404a-852f-913838c1eec0)

Therefore, the table shows that the company made positive changes within the same period in 2018 and 2019. It seems that the sales in 2018 were better than those in 2019 because the variance percentage was greater. However, in 2020 within the same period, the data indicated that the sales amount had a dropdown because of the package changes. For the long-term observation, the sales continue dropping down from 0.98 to -0.47.

* 12 weeks before and after period in 2018, 2019, and 2020.
```MySQL
WITH cte1 AS (SELECT week_number,calender_year, SUM(sales) AS total_sales
FROM clean_date
GROUP BY week_number,calender_year),
cte2 AS (SELECT calender_year,
SUM(CASE WHEN week_number BETWEEN 12 AND 23 THEN total_sales END) AS before_total_sales,
SUM(CASE WHEN week_number BETWEEN 25 AND 35 THEN total_sales END) AS after_total_sales
FROM cte1
GROUP BY calender_year)
SELECT calender_year,after_total_sales-before_total_sales AS sale_variance,
ROUND((after_total_sales-before_total_sales)/before_total_sales*100,2) AS percentage_variance
FROM cte2
ORDER BY calender_year
```
![image](https://github.com/user-attachments/assets/db693f69-42e1-42ac-b585-efbaa61f2ee9)

From 2018 to 2019, there was a substantial decline in financial performance, with an increase in losses of 159,716,572 units. This represents a worsening decline of 1.85 percentage points (from -6.57% in 2018 to -8.42% in 2019). From 2019 to 2020, the losses continued to grow, increasing by 142,521,630 units, with the percentage change worsening by another 1.72 percentage points (from -8.42% to -10.14%). Over the three years, the company or entity experienced a continuous and accelerated decline. The rate of losses is not only increasing but the percentage change is becoming more severe with each passing year. The percentage decline over the years shows a pattern of increasing financial strain or decreased sales, which could be due to external factors (such as market conditions, increased competition, or macroeconomic events) or internal challenges (e.g., operational inefficiencies, poor financial management, or declining market demand).

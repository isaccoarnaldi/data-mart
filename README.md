# 🛒 Case Study #5: Data Mart
<p align="center">
<img width="400px"  src="https://8weeksqlchallenge.com/images/case-study-designs/5.png" />
</p>

## 📚 Table of Contents
- [Business Task](#business-task)
- [Dataset](#dataset)
- [Question and Solution](#question-and-solution)

Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-5/). 

***

## Business Task
Danny seeks assistance in analyzing the sales performance of Data Mart following a major sustainability update in June 2020, which involved implementing sustainable packaging methods. He wants to quantify the impact of these changes across various business areas and determine the most affected platforms, regions, segments, and customer types. Additionally, he is interested in insights to minimize the impact on sales for future sustainability updates.

## Dataset

The case study involves a single table, `data_mart.weekly_sales`, depicted in the Entity Relationship Diagram. 

<img alt="image" src="https://8weeksqlchallenge.com/images/case-study-5-erd.png"> 		

This table encompasses international operations with a multi-region strategy, incorporating both retail and online platforms through a Shopify store. Customer segment and type data include personal age and demographics shared with Data Mart. The dataset includes columns such as `transactions` (count of unique purchases) and `sales` (dollar amount). Each record is tied to a specific aggregated weekly sales data slice, indicated by the `week_date` representing the start of the sales week. 

A sample of some of the rows is shown below:

| week_date |   region      |   platform  |   segment |   customer_type |   transactions |   sales       |
| --------- | --------------| ----------- | --------- | --------------- | -------------- | ------------- |
| 9/9/20    | OCEANIA       |   Shopify   |   C3      |   New           |   610          |   110033.89   |
| 29/7/20   | AFRICA        |   Retail    |   C1      |   New           |   110692       |   3053771.19  |
| 22/7/20   | EUROPE        |   Shopify   |   C4      |   Existing      |   24           |   8101.54     |
| 13/5/20   | AFRICA        |   Shopify   |   null    |   Guest         |   5287         |   1003301.37  |
| 24/7/19   | ASIA          |   Retail    |   C1      |   New           |   127342       |   3151780.41  |
| 10/7/19   | CANADA        |   Shopify   |   F3      |   New           |   51           |   8844.93     |
| 26/6/19   | OCEANIA       |   Retail    |   C3      |   New           |   152921       |   5551385.36  |
| 29/5/19   | SOUTH AMERICA |   Shopify   |   null    |   New           |   53           |   10056.2     |
| 22/8/18   | AFRICA        |   Retail    |   null    |   Existing      |   31721        |   1718863.58  |
| 25/7/18   | SOUTH AMERICA |   Retail    |   null    |   New           |   2136         |   81757.91    |

***

## Question and Solution

### A. Data Cleansing
````sql
DROP TABLE IF EXISTS data_mart.clean_weekly_sales;
CREATE TABLE data_mart.clean_weekly_sales AS
SELECT
  TO_DATE(week_date, 'dd/mm/yy') AS week_date,
  DATE_PART('week', TO_DATE(week_date, 'dd/mm/yy')) AS week_number,
  DATE_PART('month', TO_DATE(week_date, 'dd/mm/yy')) AS month_number,
  DATE_PART('year', TO_DATE(week_date, 'dd/mm/yy')) AS calendar_year,
  region,
  platform,
  CASE
    WHEN segment = 'null' THEN 'Unknown'
    ELSE segment
  END AS segment,
  CASE
    WHEN RIGHT(segment, 1) = '1' THEN 'Young Adults'
    WHEN RIGHT(segment, 1) = '2' THEN 'Middle Aged'
    WHEN RIGHT(segment, 1) IN ('3', '4') THEN 'Retirees'
    ELSE 'Unknown'
  END AS age_band,
  CASE
    WHEN LEFT(segment, 1) = 'C' THEN 'Couples'
    WHEN LEFT(segment, 1) = 'F' THEN 'Families'
    ELSE 'Unknown'
  END AS demographic,
  customer_type,
  transactions,
  sales,
  ROUND(
    sales :: NUMERIC / transactions,
    2
  ) AS avg_transaction
FROM data_mart.weekly_sales;

SELECT *
FROM data_mart.clean_weekly_sales
````

#### Answer (sample)
<img width="1064" alt="image" src="https://github.com/isaccoarnaldi/data-mart/blob/main/clean_table.png"> 

***

### B. Data Exploration

#### 1.  What day of the week is used for each `week_date` value?

````sql
SELECT DISTINCT TO_CHAR(week_date, 'Day') AS weekday
FROM data_mart.clean_weekly_sales;
````

**Answer**
| weekday |
|---------|
| Monday  |

#

### 2.  What range of week numbers are missing from the dataset?
- `GENERATE_SERIES` creates a series of week number
- `WHERE NOT EXISTS` to filter missing week number

````sql
SELECT *
FROM generate_series(1, 52) AS missing_week_number
WHERE NOT EXISTS(
		SELECT 1
    		FROM data_mart.clean_weekly_sales
    		WHERE missing_week_number = week_number
     );
````

**Answer**
| missing\_week\_number |
| --------------------- |
| 1                     |
| 2                     |
| 3                     |
| 4                     |
| 5                     |
| 6                     |
| 7                     |
| 8                     |
| 9                     |
| 10                    |
| 11                    |
| 12                    |
| 37                    |
| 38                    |
| 39                    |
| 40                    |
| 41                    |
| 42                    |
| 43                    |
| 44                    |
| 45                    |
| 46                    |
| 47                    |
| 48                    |
| 49                    |
| 50                    |
| 51                    |
| 52                    |

#

### 3.  How many total transactions were there for each year in the dataset?

````sql
SELECT calendar_year,
       SUM(transactions) total_transactions
FROM data_mart.clean_weekly_sales
GROUP BY 1
ORDER BY 1;

````
**Answer**
| calendar\_year | total\_transactions |
| -------------- | ------------------- |
| 2018           | 346406460           |
| 2019           | 365639285           |
| 2020           | 375813651           |

#

### 4.  What is the total sales for each region for each month?

````sql
WITH cte AS(
          SELECT
             DATE_TRUNC('month', week_date) AS start_of_month,
             region,
             SUM(sales) total_transactions,
             RANK() OVER(ORDER BY DATE_TRUNC('month', week_date)) AS rank
          FROM data_mart.clean_weekly_sales
          GROUP BY 1, 2
)

SELECT start_of_month,
       region,
       total_transactions
FROM cte
WHERE rank = 1
ORDER BY 2

````
**Answer**

*Sample for the first month of 7 regions*
| start\_of\_month | region        | total\_transactions |
| ---------------- | ------------- | ------------------- |
| 2018-03-01       | AFRICA        | 130542213           |
| 2018-03-01       | ASIA          | 119180883           |
| 2018-03-01       | CANADA        | 33815571            |
| 2018-03-01       | EUROPE        | 8402183             |
| 2018-03-01       | OCEANIA       | 175777460           |
| 2018-03-01       | SOUTH AMERICA | 16302144            |
| 2018-03-01       | USA           | 52734998            |

#

### 5.  What is the total count of transactions for each platform

````sql
SELECT platform,
       SUM(transactions)
FROM data_mart.clean_weekly_sales
GROUP BY 1
````

**Answer**
| platform | sum        |
| -------- | ---------- |
| Shopify  | 5925169    |
| Retail   | 1081934227 |

#

### 6.  What is the percentage of sales for Retail vs Shopify for each month?

````sql
SELECT DATE_TRUNC('month', week_date) AS start_of_month,
       platform,
       ROUND(100 * SUM(sales) / SUM(SUM(sales)) OVER(PARTITION BY DATE_TRUNC('month', week_date)), 2) AS percentage
FROM data_mart.clean_weekly_sales
GROUP BY 1, 2
ORDER BY 1, 2
````

**Answer**
| start\_of\_month | platform | percentage |
| ---------------- | -------- | ---------- |
| 2018-03-01       | Retail   | 97.92      |
| 2018-03-01       | Shopify  | 2.08       |
| 2018-04-01       | Retail   | 97.93      |
| 2018-04-01       | Shopify  | 2.07       |
| 2018-05-01       | Retail   | 97.73      |
| 2018-05-01       | Shopify  | 2.27       |

#

### 7.  What is the percentage of sales by demographic for each year in the dataset?
````sql
SELECT DATE_PART('year', week_date) AS year,
       demographic,
       SUM(sales) AS amount,
       ROUND(100 * SUM(sales) / SUM(SUM(sales)) OVER(PARTITION BY DATE_PART('year', week_date)), 2) AS percentage
FROM data_mart.clean_weekly_sales
GROUP BY 1, 2
ORDER BY 1, 2
````

**Answer**
| year | demographic | amount     | percentage |
| ---- | ----------- | ---------- | ---------- |
| 2018 | Couples     | 3402388688 | 26.38      |
| 2018 | Families    | 4125558033 | 31.99      |
| 2018 | Unknown     | 5369434106 | 41.63      |
| 2019 | Couples     | 3749251935 | 27.28      |
| 2019 | Families    | 4463918344 | 32.47      |
| 2019 | Unknown     | 5532862221 | 40.25      |
| 2020 | Couples     | 4049566928 | 28.72      |
| 2020 | Families    | 4614338065 | 32.73      |
| 2020 | Unknown     | 5436315907 | 38.55      |

#

### 8.  Which `age_band` and `demographic` values contribute the most to Retail sales?

````sql
--Sort by age_band only
SELECT age_band,
       ROUND(100 * SUM(sales) / SUM(SUM(sales)) OVER(), 2) AS total_sales
FROM data_mart.clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY 1
ORDER BY 2 DESC

--Sort by demographic only
SELECT demographic,
       ROUND(100 * SUM(sales) / SUM(SUM(sales)) OVER(), 2) AS total_sales
FROM data_mart.clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY 1
ORDER BY 2 DESC
  
--Sort by both age_band and demographic
SELECT age_band,
       demographic,
       ROUND(100 * SUM(sales) / SUM(SUM(sales)) OVER(), 2) AS total_sales
FROM data_mart.clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY 1, 2
ORDER BY 3 DESC
````

**Answer**
#
<a href="https://ibb.co/thP3R7g"><img src="https://i.ibb.co/9v3GPF6/Capture.png" alt="Capture" border="0"></a>

❗ **Note**
* The question is asking for the contribution to sales of 2 variables `age_band`, `demographic`. Instead of only considering both of them at the same time, we should also account for each variable separately

#

### 9.  Can we use the `avg_transaction` column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?

````sql
SELECT calendar_year,
       platform,
       SUM(sales) / SUM(transactions) AS avg_transaction_size
FROM data_mart.clean_weekly_sales
GROUP BY 1, 2
ORDER BY 1, 2
````

**Answer**
| calendar\_year | platform | avg\_transaction\_size |
| -------------- | -------- | ---------------------- |
| 2018           | Retail   | 36                     |
| 2018           | Shopify  | 192                    |
| 2019           | Retail   | 36                     |
| 2019           | Shopify  | 183                    |
| 2020           | Retail   | 36                     |
| 2020           | Shopify  | 179                    |

***                    

## Before & After Analysis

### 1.  What is the total sales for the 4 weeks before and after `2020-06-15`? What is the growth or reduction rate in actual values and percentage of sales?
First, let's identify the `week_number` value of `2020-06-15`. The query below will return `25`

````sql
SELECT DISTINCT week_number
FROM data_mart.clean_weekly_sales
WHERE week_date = '2020-06-15'
````

**The total sales**
As the package change came into effect on `2020-06-15`, we also classify `week_number` 25 into period `After` as follows 

````sql
SELECT SUM(sales) AS total_sales,
       CASE
         WHEN week_number BETWEEN 21
         AND 24 THEN 'Before'
         WHEN week_number BETWEEN 25
         AND 28 THEN 'After'
         ELSE NULL END AS period
FROM data_mart.clean_weekly_sales
WHERE calendar_year = '2020'
GROUP BY period
HAVING
  (
    CASE
      WHEN week_number BETWEEN 21
      AND 24 THEN 'Before'
      WHEN week_number BETWEEN 25
      AND 28 THEN 'After'
      ELSE NULL END
    ) IS NOT NULL
````

**Answer**
| total\_sales | period |
| ------------ | ------ |
| 2345878357   | Before |
| 2318994169   | After  |

**Next, the question asks for the difference of `sales` in actual values and in percentage between the `Before` and `After` period**
````sql
WITH cte_1 AS(
            SELECT SUM(sales) AS total_sales,
                   CASE
                     WHEN week_number BETWEEN 21
                     AND 24 THEN 'Before'
                     WHEN week_number BETWEEN 25
                     AND 28 THEN 'After'
                     ELSE NULL END AS period
             FROM data_mart.clean_weekly_sales
             WHERE calendar_year = '2020'
             GROUP BY period
),

cte_2 AS(
       SELECT *,
             (total_sales - LAG(total_sales, 1) OVER()) AS sale_diff,
             ROUND(100 * (
  	        total_sales - LAG(total_sales, 1) OVER()) :: NUMERIC / LAG(total_sales, 1)
                OVER(), 2) AS percentage_of_change
       FROM cte_1
       WHERE period IS NOT NULl
)

SELECT sale_diff AS rate,
       percentage_of_change
FROM cte_2
WHERE sale_diff IS NOT NULL
````

**Answer**
| sale_diff | reduction_rate |
| --------- | -------------- |
| -26884188 | -1.15          |

#

### 2.  What about the entire 12 weeks before and after?

````sql
WITH cte_1 AS(
            SELECT SUM(sales) AS total_sales,
                   CASE
                     WHEN week_number BETWEEN 13
      		     AND 24 THEN 'Before'
      		     WHEN week_number BETWEEN 25
      		     AND 36 THEN 'After'
      		     ELSE NULL END AS period
  	   FROM data_mart.clean_weekly_sales
  	   WHERE calendar_year = '2020'
  	   GROUP BY period
),

cte_2 AS(
        SELECT *,
              (total_sales - LAG(total_sales, 1) OVER()) AS sale_diff,
              ROUND(100 * (
	         total_sales - LAG(total_sales, 1) OVER()) :: NUMERIC / LAG(total_sales, 1)
                 OVER(), 2) AS percentage_of_change
        FROM cte_1
        WHERE period IS NOT NULl
)

SELECT sale_diff AS rate,
       percentage_of_change
FROM cte_2
WHERE sale_diff IS NOT NULL
````

**Answer**
| sale_diff  | percentage_of_change |
| ---------- | -------------------- |
| -152325394 | -2.14                |

#

### 3.  How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?
In other words, assuming that in 2018 and 2019, the same period of package change was applied, what the rate of percentage of change would be like for `sale` metric 

````sql
WITH cte_1 AS(
            SELECT calendar_year,
                   SUM(sales) AS total_sales,
    		   CASE
		     WHEN week_number BETWEEN 21
     		     AND 24 THEN 'Before'
      		     WHEN week_number BETWEEN 25
     		     AND 28 THEN 'After'
     		     ELSE NULL END AS period
  	   FROM data_mart.clean_weekly_sales
  	   GROUP BY 1, 3
  	   ORDER BY 1, 3 DESC
),

cte_2 AS(
       SELECT *,
             (total_sales - LAG(total_sales, 1) OVER(PARTITION BY calendar_year)) AS rate,
             ROUND(100 * (
               total_sales - LAG(total_sales, 1) OVER(PARTITION BY calendar_year)
               ) :: NUMERIC / LAG(total_sales, 1) OVER(PARTITION BY calendar_year),
               2) AS percentage_of_change
       FROM cte_1
       WHERE period IS NOT NULL
)

SELECT calendar_year,
       rate,
       percentage_of_change
FROM cte_2
WHERE rate IS NOT NULL

````
**Answer**
| calendar\_year | rate       | percentage\_of\_change |
| -------------- | ---------- | ---------------------- |
| 2018           | 4102105    | 0.19                   |
| 2019           | 2336594    | 0.10                   |
| 2020           | \-26884188 | \-1.15                 |

***

## D. Bonus Question

Areas of the business having the highest negative impact in sales metrics performance in 2020 for the 12 week before and after period:

* **`region`**

````sql
	WITH cte_1 AS(
	  SELECT region,
	         CASE
	           WHEN week_number BETWEEN 13
	           AND 24 THEN 'Before'
	           WHEN week_number BETWEEN 25
	           AND 36 THEN 'After'
	           ELSE NULL END AS period,
	         SUM(sales) AS total_sales
	  FROM data_mart.clean_weekly_sales
	  WHERE calendar_year = '2020'
	  GROUP BY 1, 2
	  ORDER BY 1, 2 DESC
	),

	cte_2 AS(
	       SELECT *,
	             (total_sales - LAG(total_sales, 1) OVER(PARTITION BY region)) AS sale_diff,
	             ROUND(100 * (
		          total_sales - LAG(total_sales, 1) OVER(PARTITION BY region)) :: NUMERIC / LAG(total_sales, 1) OVER(PARTITION BY region),
	                 2) AS percentage_of_change
	        FROM cte_1
	)
	SELECT region,
	       sale_diff AS rate,
	       percentage_of_change
	FROM cte_2
	WHERE sale_diff IS NOT NULL
	ORDER BY 3

````

**Answer**
| region        | rate       | percentage\_of\_change |
| ------------- | ---------- | ---------------------- |
| ASIA          | \-53436845 | \-3.26                 |
| OCEANIA       | \-71321100 | \-3.03                 |
| SOUTH AMERICA | \-4584174  | \-2.15                 |
| CANADA        | \-8174013  | \-1.92                 |
| USA           | \-10814843 | \-1.60                 |
| AFRICA        | \-9146811  | \-0.54                 |
| EUROPE        | 5152392    | 4.73                   |


Similarly, we execute the above query for each field and get the following results
*   `platform`

| platform | rate        | percentage\_of\_change |
| -------- | ----------- | ---------------------- |
| Retail   | \-168083834 | \-2.43                 |
| Shopify  | 15758440    | 7.18                   |

*   `age_band`

| age\_band    | rate       | percentage\_of\_change |
| ------------ | ---------- | ---------------------- |
| Unknown      | \-92393021 | \-3.34                 |
| Middle Aged  | \-22994292 | \-1.97                 |
| Retirees     | \-29549521 | \-1.23                 |
| Young Adults | \-7388560  | \-0.92                 |

*   `demographic`

| demographic | rate       | percentage\_of\_change |
| ----------- | ---------- | ---------------------- |
| Unknown     | \-92393021 | \-3.34                 |
| Families    | \-42320015 | \-1.82                 |
| Couples     | \-17612358 | \-0.87                 |

*   `customer_type`

| customer\_type | rate       | percentage\_of\_change |
| -------------- | ---------- | ---------------------- |
| Guest          | \-77202666 | \-3.00                 |
| Existing       | \-83872973 | \-2.27                 |
| New            | 8750245    | 1.01                   |

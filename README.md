# Walmart sales data analysis using SQL
<img width="320" height="320" alt="walmart-supercentre-logo-png_seeklogo-505325" src="https://github.com/user-attachments/assets/c975f770-58d7-4ad0-9a78-7b65f02a68e1" />

📊 Walmart Sales Analysis

📌 Overview:
This project involves a comprehensive analysis of Walmart’s weekly sales data using SQL. The goal is to uncover key business insights related to seasonality, growth trends, and correlations with external factors (fuel prices). This README outlines the objectives, dataset schema, business problems, SQL solutions, findings, and conclusions.

🎯 Objectives
- Calculate the correlation between fuel prices and weekly sales.
- Identify the month with the strongest correlation (positive or negative).
- Compute year-over-year (YoY) growth for each month and find the most consistent growth month.
- Perform a seasonal analysis:
  + Detect seasonal peaks and troughs.
  + Calculate the Seasonality Index (monthly sales ÷ annual average sales).

📂 Dataset
The dataset comes from Walmart Sales Data.

## Schema
```sql
DROP TABLE IF EXISTS walmart_sales;
CREATE TABLE walmart_sales
(
    store_id       INT,
    dept_id        INT,
    Date           DATE,
    Weekly_Sales   DECIMAL(10,2),
    Fuel_Price     DECIMAL(5,2),
    Temperature    DECIMAL(5,2),
    CPI            DECIMAL(6,2),
    Unemployment   DECIMAL(5,2)
);
```

🔎 Business Problems and Solutions
1.Calculate total sales (Weekly_Sales) by each month in 2010. 
```sql
SELECT 
    DATE_FORMAT(STR_TO_DATE(Date, '%d-%m-%Y'), '%m') AS Month,
    ROUND(SUM(Weekly_Sales), 2) AS Total_Sales
FROM walmart_sales
WHERE DATE_FORMAT(STR_TO_DATE(Date, '%d-%m-%Y'), '%Y') = '2010'
GROUP BY DATE_FORMAT(STR_TO_DATE(Date, '%d-%m-%Y'), '%m')
ORDER BY Total_Sales DESC;
```
2.Compare average sales between holiday weeks (Holiday_Flag = 1) and regular weeks (Holiday_Flag = 0). Calculate the percentage difference.
```sqlsql
SELECT 
    AVG(CASE WHEN Holiday_Flag = 1 THEN Weekly_Sales END) AS Holiday_Avg,
    AVG(CASE WHEN Holiday_Flag = 0 THEN Weekly_Sales END) AS Regular_Avg,
    (
      (AVG(CASE WHEN Holiday_Flag = 1 THEN Weekly_Sales END) - 
       AVG(CASE WHEN Holiday_Flag = 0 THEN Weekly_Sales END)
      ) / AVG(CASE WHEN Holiday_Flag = 0 THEN Weekly_Sales END)
    ) * 100 AS Percentage_Difference
FROM walmart_sales;
```
3. Divide data into 4 quarters and calculate total sales for each quarter. Which quarter has the highest growth rate compared to the previous quarter?
```sqlsql
WITH Quarter_Sales AS (
    SELECT 
        YEAR(STR_TO_DATE(Date, '%d-%m-%Y')) AS Year,
        QUARTER(STR_TO_DATE(Date, '%d-%m-%Y')) AS Quarter,
        SUM(Weekly_Sales) AS Total_Sales
    FROM walmart_sales
    GROUP BY 
        YEAR(STR_TO_DATE(Date, '%d-%m-%Y')),
        QUARTER(STR_TO_DATE(Date, '%d-%m-%Y'))
),
Growth AS (
    SELECT 
        Year,
        Quarter,
        Total_Sales,
        ( (Total_Sales - LAG(Total_Sales) OVER (PARTITION BY Year ORDER BY Quarter)) 
          / LAG(Total_Sales) OVER (PARTITION BY Year ORDER BY Quarter) ) * 100 AS Growth_Rate_Percent
    FROM Quarter_Sales
)
SELECT *
FROM Growth
WHERE Growth_Rate_Percent IS NOT NULL
ORDER BY Growth_Rate_Percent DESC
LIMIT 1;
```
4. Find the top 5 weeks with highest sales in the entire dataset. Display date, sales amount, and whether it's a holiday week.
```sql
SELECT 
   STR_TO_DATE(Date, '%d-%m-%Y') AS Week_Date,
   MAX(holiday_flag) AS holiday_week,
   SUM(weekly_sales) AS total_sales
FROM walmart_sales
GROUP BY STR_TO_DATE(Date, '%d-%m-%Y')
ORDER BY total_sales DESC 
LIMIT 5; 
```
5.Categorize temperature into ranges (Cold: <50°F, Moderate: 50-70°F, Hot: >70°F) and calculate average sales for each temperature range
```sqlsql
SELECT 
    CASE 
        WHEN Temperature < 50 THEN 'Cold'
        WHEN Temperature BETWEEN 50 AND 70 THEN 'Moderate'
        ELSE 'Hot'
    END AS Temp_Range,
    AVG(Weekly_Sales) AS Avg_Sales
FROM walmart_sales
GROUP BY Temp_Range
ORDER BY Avg_Sales DESC;
```
6.Calculate 4-week moving average of sales for each store. Compare with actual sales to identify weeks with unusual fluctuations
```sql
WITH store_weekly AS (
SELECT 
	store,
    STR_TO_DATE(Date, '%d-%m-%y') AS week_date,
    SUM(weekly_sales) AS total_sales
FROM walmart_sales
GROUP BY store, STR_TO_DATE(Date, '%d-%m-%y'))
SELECT 
	store,
    week_date,
    total_sales,
    ROUND(AVG(total_sales) OVER(
    PARTITION BY store
    ORDER BY week_date
    ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_4weeks,
    ROUND(
		total_sales - AVG(total_sales) OVER(
    PARTITION BY store
    ORDER BY week_date
    ROWS BETWEEN 3 PRECEDING AND CURRENT ROW), 2) AS Difference 
FROM store_weekly
ORDER BY store, total_sales;
```
7.Analyze sales by week within each month (week 1, 2, 3, 4 of each month). Which week of the month typically has the highest sales?
```sql
WITH sales_with_week AS (
    SELECT 
        STR_TO_DATE(Date, '%d-%m-%Y') AS dt,
        Weekly_Sales,
        YEAR(STR_TO_DATE(Date, '%d-%m-%Y')) AS yr,
        MONTH(STR_TO_DATE(Date, '%d-%m-%Y')) AS mn,
        CEIL(DAY(STR_TO_DATE(Date, '%d-%m-%Y')) / 7) AS week_of_month
    FROM walmart_sales
)
SELECT 
    yr AS year,
    mn AS month,
    week_of_month,
    SUM(Weekly_Sales) AS total_sales,
    AVG(Weekly_Sales) AS avg_sales
FROM sales_with_week
GROUP BY yr, mn, week_of_month
ORDER BY yr, mn, week_of_month;
```
8.Calculate correlation coefficient between Fuel_Price and Weekly_Sales by month. Which month has the strongest correlation (positive or negative)?
```sql
SELECT 
    MONTH(STR_TO_DATE(Date, '%d-%m-%Y')) AS Month,
    ( (AVG(Fuel_Price * Weekly_Sales) - AVG(Fuel_Price) * AVG(Weekly_Sales)) /
      (STDDEV(Fuel_Price) * STDDEV(Weekly_Sales)) ) AS Correlation
FROM walmart_sales
GROUP BY MONTH(STR_TO_DATE(Date, '%d-%m-%Y'))
ORDER BY ABS(Correlation) DESC;
```
9.If dataset spans multiple years, calculate year-over-year sales growth for the same month. Which week shows the most consistent growth?
```sql
WITH t1 AS (
    SELECT 
        MONTH(STR_TO_DATE(Date, '%d-%m-%Y')) AS Month,
        YEAR(STR_TO_DATE(Date, '%d-%m-%Y')) AS Year,
        SUM(weekly_sales) AS total_sales
    FROM walmart_sales
    GROUP BY YEAR(STR_TO_DATE(Date, '%d-%m-%Y')), MONTH(STR_TO_DATE(Date, '%d-%m-%Y'))
),

yoy_growth AS (
    SELECT 
        cur.Month,
        cur.Year,
        cur.Total_sales,
        ((cur.Total_sales - prev.Total_sales) / prev.Total_sales) * 100 AS growth_percent
    FROM t1 cur
    JOIN t1 prev 
      ON cur.Month = prev.Month
     AND cur.Year = prev.Year + 1
)

SELECT 
    Month,
    AVG(growth_percent) AS Avg_Growth,
    STDDEV(growth_percent) AS Growth_Variability
FROM yoy_growth
GROUP BY Month
ORDER BY Growth_Variability ASC;
```
10.Create a comprehensive seasonal analysis report showing:
- Identify seasonal peaks and troughs throughout the year
- Calculate seasonality index for each month (month sales / annual average sales)
```sqlsql
WITH monthly_sales AS (
SELECT 
	MONTH(STR_TO_DATE(Date, '%d-%m-%Y')) AS Month,
    Year(STR_TO_DATE(Date, '%d-%m-%Y')) AS Year,
    SUM(weekly_sales) AS total_monthly_sales
FROM walmart_sales
GROUP BY 
		MONTH(STR_TO_DATE(Date, '%d-%m-%Y')),
		Year(STR_TO_DATE(Date, '%d-%m-%Y'))
),
annual_avg AS (
SELECT
	Year,
    AVG(total_monthly_sales) AS annual_avg_sales
FROM monthly_sales
GROUP BY year
),
seasonality AS (
SELECT 
	m.month,
    m.year,
    m.total_monthly_sales,
    a.annual_avg_sales,
    ROUND(m.total_monthly_sales / a.annual_avg_sales, 2) AS seasonality_index
FROM monthly_sales m
JOIN annual_avg a
ON m.year = a.year
)
SELECT 
	month,
    year,
    total_monthly_sales,
    annual_avg_sales,
    seasonality_index,
    CASE 
		WHEN seasonality_index > 1.1 THEN 'Peak'
        WHEN seasonality_index < 0.0 THEN 'Trough'
        ELSE 'Normal'
	END AS Seasonality_flag
FROM seasonality
ORDER BY year, month;
FROM seasonality
```
📊 Findings and Conclusion
- Fuel Price Correlation: Certain months show a strong negative correlation, indicating higher fuel prices may reduce consumer spending.
- YoY Growth: Some months (e.g., holiday season) demonstrate consistent growth, while others fluctuate heavily.
- Seasonality:
  + Peaks: November–December (holiday shopping).
  + Troughs: Summer months show weaker demand.
  + Seasonality Index quantifies sales uplift compared to the annual average.

✍️ Author
Van Huu Hien Nguyen – Business Analytics Student
This project is part of my SQL portfolio, showcasing skills in data cleaning, analysis, and business insights generation.
Feel free to reach out for feedback or collaboration!




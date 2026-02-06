# Shopping-Sales-Trends-EDA-
Fictional retail sales analysis conducted with SQL and Tableau querying/calculations.

- Using the Chart Swap filter, change between Default View (product Color and Age Cluster segmentation) and Map View (States drilldown information).

![](https://github.com/BrenoEnrico-MDSantos/Shopping-Sales-Trends-EDA-/blob/main/SalesPNG1.png)
Default view with unfiltered values.

![](https://github.com/BrenoEnrico-MDSantos/Shopping-Sales-Trends-EDA-/blob/main/SalesPNG2.png)
Specific filtering on second chart view.

# Data source
- Original file "shopping_trends_updated.csv" from "https://www.kaggle.com/datasets/iamsouravbanerjee/customer-shopping-trends-dataset
- The original columns and data types were:
  ```
  Customer ID	int
  Age	int
  Gender	text
  Item Purchased	text
  Category	text
  Purchase Amount (USD)	int
  Location	text
  Size	text
  Color	text
  Season	text
  Review Rating	double
  Subscription Status	text
  Shipping Type	text
  Discount Applied	text
  Promo Code Used	text
  Previous Purchases	int
  Payment Method	text
  Frequency of Purchases	text
  ```
- Must be noted: date fields, which would be vital for a time series analysis; no stockage values across unities, production costs or churn rate. The data is very homogenous and even across dimensions (Gender, Age segments, subscribers etc.), hence the "trends" don't pop up as they normally would.
- I tested linear regression models on tableau, and the correlations don't say much. So, most of the work in both tools were to conduct a basic wrangling and display of what's available.

# SQL Usage 

- Initial querying and alteration; further changes were made in Tableau as I understood the data better.
- 
```
-- With data scheme created and the csv imported on it, the table was named "shopping".

USE data

SELECT `Customer ID` FROM shopping GROUP BY `Customer ID` HAVING COUNT(*) > 1; -- Always remember the backticks(``) on elements with spaces

select distinct(Location) FROM shopping ORDER BY Location ASC;

SELECT DISTINCT(Location), SUM(`Purchase Amount (USD)`) OVER(PARTITION BY `Location`) AS Per_state 
FROM shopping ORDER BY Per_state DESC;

SELECT DISTINCT(`Item Purchased`), Size, COUNT(`Item Purchased`) 
OVER(PARTITION BY `Item Purchased`) as ovo 
FROM shopping order by ovo;

-- Created procedure for quick limited SELECT:

DELIMITER $$
CREATE PROCEDURE head(IN rows_ INT) 
	BEGIN
		SELECT * FROM shopping LIMIT rows_;
    END $$
DELIMITER ;

CALL head(5);

--Outlier detection with Z Score method: (x - AVG)/Standard Deviation resulting in an acceptable Confidence Interval for both Payment and Age columns.

WITH USDOutliers AS 
	(SELECT `Purchase Amount (USD)`, 
	(`Purchase Amount (USD)` - AVG(`Purchase Amount (USD)`) OVER())/ STD(`Purchase Amount (USD)`) OVER() AS zscore 
	FROM shopping)
SELECT * FROM USDOutliers WHERE zscore BETWEEN -2 AND 2 ORDER BY zscore DESC;

WITH AgeOutliers AS 
	(SELECT Age, 
	(Age - AVG(Age) OVER())/ STD(Age) OVER() AS zscore 
	FROM shopping)
SELECT * FROM AgeOutliers ORDER BY zscore DESC;

SELECT
	DISTINCT(Location),
    COUNT(`Customer ID`) OVER(PARTITION BY Location) AS Cont,
    SUM(`Purchase Amount (USD)`) OVER(PARTITION BY Location) AS SumPay,
    MAX(`Purchase Amount (USD)`) OVER(PARTITION BY Location) AS MaxPay, 
    MIN(`Purchase Amount (USD)`) OVER(PARTITION BY Location) AS MinPay,
    SUM(`Purchase Amount (USD)`) OVER() AS TotalPay,
    COUNT(`Purchase Amount (USD)`) OVER() AS TotalCount
FROM shopping ORDER BY Cont DESC;

SELECT 
	DISTINCT(`Frequency of Purchases`) AS Frequency,
	`Subscription Status`,
    ROUND(AVG(`Purchase Amount (USD)`) OVER(PARTITION BY `Frequency of Purchases`, `Subscription Status`),2) AS Average,
    COUNT(`Customer ID`) OVER(PARTITION BY `Frequency of Purchases`, `Subscription Status`) AS Cont,
    COUNT(`Frequency of Purchases`) OVER() AS TotalCont
FROM shopping ORDER BY `Subscription Status` DESC;

SELECT 
	DISTINCT(Season),
    SUM(`Purchase Amount (USD)`) OVER(PARTITION BY Season) AS SumSeason,
    COUNT(`Purchase Amount (USD)`) OVER(PARTITION BY Season) AS ContSeason,
    Category,
	COUNT(`Purchase Amount (USD)`) OVER(PARTITION BY Season, Category) AS ContBoth,
    SUM(`Purchase Amount (USD)`) OVER(PARTITION BY Category, Season) AS SumCat,
    SUM(`Purchase Amount (USD)`) OVER() AS TotalSum
FROM shopping ORDER BY SumSeason DESC, SumCat DESC;

-- Cluster column created to later view how each segment would behave with multi filtering:

ALTER TABLE shopping ADD COLUMN Age_Clusters VARCHAR(12);

UPDATE shopping SET Age_Clusters = 
	(CASE
		WHEN Age <20 THEN "<20s"
        WHEN Age BETWEEN 20 AND 30 THEN "20s"
        WHEN Age BETWEEN 30 AND 40 THEN "30s"
        WHEN Age BETWEEN 40 AND 50 THEN "40s"
        WHEN Age BETWEEN 50 AND 60 THEN "50s"
        WHEN Age BETWEEN 60 AND 70 THEN "60s"
        ELSE ">70s"
    END);

SELECT Gender, Age, (CASE 
	WHEN Gender = "Male" THEN "M"
    ELSE "F"
    END) AS Gender_Abbr
FROM shopping ORDER BY Age DESC;

SELECT DISTINCT(Age_Clusters), Gender,
	COUNT(`Purchase Amount (USD)`) OVER(PARTITION BY Age_Clusters, Gender) AS CountPurchases,
	CAST(SUM(`Purchase Amount (USD)`) OVER(PARTITION BY Age_Clusters, Gender) AS FLOAT) AS `$SumPurchases`,
    ROUND(SUM(`Purchase Amount (USD)`) OVER(PARTITION BY Age_Clusters, Gender) / SUM(`Purchase Amount (USD)`) OVER() * 100, 2) AS `$Purchases(%)`,
    ROUND(AVG(`Purchase Amount (USD)`) OVER(PARTITION BY Age_Clusters, Gender), 2) AS AvgSpending
FROM shopping ORDER BY `$Purchases(%)` DESC;

SELECT DISTINCT(`Discount Applied`), `Subscription Status`, `Promo Code Used`, `Frequency of Purchases`,
  COUNT(*) over(PARTITION BY `Discount Applied`, `Subscription Status`, `Promo Code Used`,`Frequency of Purchases`) AS Count
FROM shopping ORDER BY `Frequency of Purchases`;
```

# Tableau Usage

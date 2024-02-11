# Reporting Sales' Numbers using Advanced SQL in BigQuery

## Overview
Applying window functions, subqueries, and Common Table Expressions (CTEs) in BigQuery, this  project adeptly extracted monthly sales figures, encompassing mean tax rates and the percentage of provinces with tax, categorized by country and region.

## Project Structure
The final report was creted in an orderly manner with different subqueires:
+ Query monthly sales numbers in each Country & region. Include in the query a number of orders, customers and sales persons in each month with a total amount with tax earned.
+ Calculate cumulative sum of the total amount with tax earned per country & region.
+ Add ‘sales_rank’ column that ranks rows from best to worst for each country based on total amount with tax earned each month.
+ Add taxes on a country level:
  * As taxes can vary in country based on province, add a column for average tax rate in a country.
  * As not all regions have data on taxes, be transparent and add a column representing the percentage of provinces with available tax rates for each country


## Tools Used
BigQuery

## Data Source
AdventureWorks 2001-2004

## Analysis

```sql
WITH
  monthly_sales AS (
  SELECT
    LAST_DAY(DATE(salesorder.OrderDate)) AS order_month,
    sales_territory.CountryRegionCode,
    sales_territory.Name AS Region,
    COUNT(DISTINCT salesorder.SalesOrderID) AS number_orders,
    COUNT(DISTINCT salesorder.CustomerID) AS number_customers,
    COUNT(DISTINCT(salesorder.SalesPersonID)) AS no_salesPersons,
    --ROUND(SUM(salesorder.TotalDue)) AS Total_w_tax,
    CAST(SUM(salesorder.TotalDue) AS INT) AS Total_w_tax
  FROM
    `adwentureworks_db.salesorderheader` AS salesorder 
  INNER JOIN
    `adwentureworks_db.salesterritory` AS sales_territory
  ON
    salesorder.TerritoryID=sales_territory.TerritoryID
  GROUP BY
    LAST_DAY(DATE(salesorder.OrderDate)),
    sales_territory.CountryRegionCode,
    sales_territory.Name),
  /*Subquery with CTE is for calculating mean tax rate and percent of provinces with taxes in each country*/
  provinceperct AS(
  WITH 
  /*provinces query is for count of provinces per country*/
    provinces AS(
    SELECT
      p.CountryRegionCode AS country,
      COUNT (p.StateProvinceID) AS count_provinces
    FROM
      `adwentureworks_db.stateprovince` AS p
    GROUP BY
      p.CountryRegionCode),
  /*provincestax combines tax rate table with provinces to give a count of provinces with tax per country*/
    provinceswtax AS (
    SELECT
      province.CountryRegionCode,
      COUNT (DISTINCT taxrate.StateProvinceID) AS count_provinces_w_tax,
      ROUND(AVG(taxrate.TaxRate),1) AS mean_tax_rate
    FROM
      `adwentureworks_db.stateprovince` AS province
    JOIN
      `adwentureworks_db.salestaxrate` AS taxrate
    ON
      taxrate.StateProvinceID=province.StateProvinceID
    GROUP BY
      province.CountryRegionCode)
  SELECT
    provinces.country,
    provinceswtax.mean_tax_rate,
    ROUND((provinceswtax.count_provinces_w_tax)/(provinces.count_provinces),2) AS perc_provinces_w_tax,
  FROM
    provinces
  JOIN
    provinceswtax
  ON
    provinces.country=provinceswtax.CountryRegionCode)
SELECT
  monthly_sales.order_month,
  monthly_sales.CountryRegionCode,
  monthly_sales.Region,
  monthly_sales.number_orders,
  monthly_sales.number_customers,
  monthly_sales.no_salesPersons,
  monthly_sales.Total_w_tax,
  RANK() OVER (PARTITION BY monthly_sales.CountryRegionCode ORDER BY Total_w_tax DESC ) AS country_sales_rank,
  SUM(Total_w_tax) OVER (PARTITION BY monthly_sales.CountryRegionCode, monthly_sales.Region ORDER BY monthly_sales.order_month ) AS cumulative_sum,
  provinceperct.mean_tax_rate,
  provinceperct.perc_provinces_w_tax
FROM
  monthly_sales,
  provinceperct
WHERE
  monthly_sales.CountryRegionCode=provinceperct.country
ORDER BY
  monthly_sales.Region DESC;
```

## Results
[Result table filtered for Canada]![sql result](https://github.com/ammu993/Advanced-SQL/assets/74145869/4f6c991c-5970-4509-a143-747ac3abcd22)





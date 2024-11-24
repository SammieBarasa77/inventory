# Data Project
![Logo](https://github.com/SammieBarasa77/ab_testing/blob/main/assets/images/Screenshot%202024-11-23%20222057.png)

Table of Contents
- [Introduction](#introduction)  
  - [1.1 Problem Statement](#11-problem-statement)  
- [Data Understanding and Initial Exploration](#data-understanding-and-initial-exploration)  
  - [2.1 Dataset Import and Setup](#21-dataset-import-and-setup)  
  - [2.2 Dataset Exploration](#22-dataset-exploration)  
- [Inventory Stock Analysis](#inventory-stock-analysis)  
  - [3.1 Current Stock Levels](#31-current-stock-levels)  
  - [3.2 Days of Supply Analysis](#32-days-of-supply-analysis)  
- [Demand Forecasting](#demand-forecasting)
  - [4.1 Sales Trends](#41sales-trends)
- [Inventory Turnover Analysis](#inventory-turnover-analysis)  
  - [5.1 Turnover Rate](#51-turnover-rate)   
- [Reorder Point Analysis](#reorder-point-analysis)  
  - [6.1 Average Daily Sales ](#61-average-daily-sales)
  - [6.2 Shipping Performance](#62-shipping-performance)
  - [6.3 Profitability Analysis ](#63-profitability-analysis)  
  - [6.4 ABC Analysis](#64-abc-analysis)  
- [Stockout and Overstock Analysis](#stockout-and-overstock-analysis)  
  - [7.1 Products with consistent stockouts](#81-products-with-consistent-stockouts)  
  - [7.2 Overstocked products](#72-overstocked-products)  
- [Backorder Analysis](#backorder-analysis)  
  - [8.1 Impacts of Backorders](#81-impacts-of-backordersy)
- [Inventory Cost Analysis](#inventory-cost-analysis)
  - [9.1 Inventory Carrying Cost](#91-inventory-carrying-cost)
- [Customer Demand Analysis](#81-customer-demand-analysis)
  - [Customer Purchasing behavior](#81-customer-purchasing-behavior) 
- [Findings, Recommendations, Insights, and Conclusion](#findings-recommendations-insights-and-conclusion)

## Introduction
### Problem Statement

Inventory management is a critical challenge for businesses aiming to maintain the right stock levels, reduce excess inventory, avoid stockouts, and optimize profitability. Poor inventory practices lead to financial losses, customer dissatisfaction, and inefficiencies. This project aims to provide actionable insights through data analysis to improve inventory processes and decision-making.

## Data Understanding and Initial Exploration
### Dataset Import and Setup
Import the dataset into a MySQL database.
Create a table to store the data.

### Dataset Exploration
Basic SELECT queries to inspect table structure, columns, data types, and content.
Identifying missing values, duplicates, and compute basic statistics (e.g., min, max, mean).

## Inventory Stock Analysis
### Current Stock Levels
Query to calculate the current stock levels for each product.
```sql
SELECT 
    Product_ID, 
    Product_Name, 
    SUM(Quantity) AS current_stock 
FROM inventory.inventory_data_1
GROUP BY Product_ID, Product_Name;
```
### Days of Supply Analysis
Calculate the days of supply based on current stock levels and average daily sales using window functions.
```sql
WITH daily_sales AS (
    SELECT 
        Product_ID, 
        SUM(sales) / COUNT(DISTINCT Order_Date) AS avg_daily_sales
    FROM inventory.inventory_data_1
    GROUP BY Product_ID, Product_Name
)
SELECT 
    i.Product_ID,
    i.Product_Name,
    i.current_stock,
    ds.avg_daily_sales,
    i.current_stock / ds.avg_daily_sales AS days_of_supply
FROM 
    (SELECT product_id, product_name, SUM(quantity) AS current_stock FROM inventory.inventory_data_1 GROUP BY Product_ID, Product_Name) i
JOIN daily_sales ds ON i.product_id = ds.product_id;
```
## Demand Forecasting
### Sales Trends
Use rolling averages with window functions to analyze sales trends over time.
```sql
-- Moving average of sales over the last 7 days
SELECT 
    Product_ID,
    Product_Name,
    Order_Date,
    AVG(Sales) OVER (PARTITION BY Product_ID ORDER BY Order_Date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg_sales
FROM inventory.inventory_data_1;
```
## Inventory Turnover Analysis
### Turnover Rate
```sql
-- Invetory turnover 
SELECT 
    Product_ID,
    Product_Name,
    SUM(Sales * 2) / NULLIF(AVG(quantity), 0) AS turnover_rate  -- Turnover rate = sales * cost / quantity because the dataset had no cost field
FROM inventory.inventory_data_1
GROUP BY Product_ID, Product_Name;
```
## Re-Order Point Analysis
### Average Daily Sales 
```sql
-- CTE to calculate the average daily sales for each product
WITH daily_sales AS (
    SELECT 
        Product_ID,
        AVG(sales) AS avg_daily_sales
    FROM inventory.inventory_data_1
    GROUP BY Product_ID
),
order_time AS (                        -- CTE to calculate the average lead time for each supplier
    SELECT 
        Order_ID,
        AVG(DATEDIFF(NOW(), Order_Date)) AS avg_order_time
    FROM inventory.inventory_data_1
    GROUP BY Order_ID
)                                       -- Calculate reorder point by joining the two CTEs
SELECT 
    i.Product_ID,
    i.Product_Name,
    ds.avg_daily_sales,
    lt.avg_order_time,
    (ds.avg_daily_sales * lt.avg_order_time) AS reorder_point
FROM daily_sales ds
JOIN inventory.inventory_data_1 i ON ds.Product_ID = i.Product_ID
JOIN order_time lt ON i.Order_ID = lt.Order_ID
GROUP BY 
    i.Product_ID, 
    i.Product_Name, 
    ds.avg_daily_sales, 
    lt.avg_order_time;
```
### Shipping Performance (Supplier performance)
```sql
SELECT 
    Ship_Mode,
    AVG(DATEDIFF(NOW(), Ship_Date)) AS avg_lead_time,
    COUNT(CASE WHEN quantity = sales THEN 1 END) / COUNT(*) * 100 AS fulfillment_rate
FROM inventory.inventory_data_1
GROUP BY Ship_Mode;
```
### Profitability Analysis 
```sql
SELECT 
    Product_ID,
    Product_Name,
    (SUM(Sales * price) - SUM(sales * cost)) / SUM(Sales * price) AS gross_margin
FROM inventory.inventory_data_1
GROUP BY Product_ID, Product_Name;
```
### ABC Analysis 
```sql
WITH product_sales AS (
    SELECT 
        Product_ID, 
        Product_Name, 
        SUM(Sales * Quantity) AS total_sales
    FROM inventory.inventory_data_1
    GROUP BY Product_ID, Product_Name
),
cumulative_sales AS (
    SELECT 
        ps.Product_ID,
        ps.Product_Name,
        ps.total_sales,
        SUM(ps.total_sales) OVER (ORDER BY ps.total_sales DESC) / (SELECT SUM(total_sales) FROM product_sales) AS cumulative_share
    FROM product_sales ps
)
SELECT 
    Product_ID,
    Product_Name,
    CASE 
        WHEN cumulative_share <= 0.8 THEN 'A'
        WHEN cumulative_share <= 0.95 THEN 'B'
        ELSE 'C'
    END AS abc_classification
FROM cumulative_sales;
```
## Stockout and Overstock Analysis
### Products with consistent stockouts
```sql
WITH stock_status AS (
    SELECT 
        Product_ID, 
        Product_Name, 
        Quantity,
        SUM(Sales) OVER (PARTITION BY Product_ID ORDER BY Order_Date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS total_sales_last_7_days
    FROM inventory.inventory_data_1
),
stockout_products AS (
    SELECT 
        Product_ID, 
        Product_Name,
        COUNT(CASE WHEN Quantity = 0 AND total_sales_last_7_days > 0 THEN 1 END) AS stockout_count
    FROM stock_status
    GROUP BY Product_ID, Product_Name
)
SELECT 
    Product_ID,
    Product_Name,
    stockout_count
FROM stockout_products
WHERE stockout_count > 0
ORDER BY stockout_count DESC;
```
### Overstocked products
```sql
WITH avg_sales AS (
    SELECT 
        Product_ID, 
        AVG(Sales) AS avg_sales_per_day
    FROM inventory.inventory_data_1
    GROUP BY Product_ID
)
SELECT 
    i.Product_ID,
    i.Product_Name,
    SUM(i.Quantity) AS total_stock,
    a.avg_sales_per_day,
    (SUM(i.Quantity) / a.avg_sales_per_day) AS estimated_days_of_inventory
FROM inventory.inventory_data_1 i
JOIN avg_sales a ON i.Product_ID = a.Product_ID
GROUP BY i.Product_ID, i.Product_Name, a.avg_sales_per_day
HAVING estimated_days_of_inventory > 180 -- Over 6 months of inventory
ORDER BY estimated_days_of_inventory DESC;
```
## Backorder Analysis
### Impacts of Backorders
```sql
-- Query to calculate the frequency and impact of backorders:
WITH backorder_info AS (
    SELECT 
        Product_ID,
        Product_Name,
        COUNT(CASE WHEN Quantity < Sales THEN 1 END) AS backorder_count,
        SUM((Sales - Quantity) * price) AS estimated_lost_revenue
    FROM inventory.inventory_data_1
    GROUP BY Product_ID, Product_Name
)
SELECT 
    Product_ID,
    Product_Name,
    backorder_count,
    estimated_lost_revenue
FROM backorder_info
WHERE backorder_count > 0
ORDER BY estimated_lost_revenue DESC;
```
## Supplier Risk Analysis
### Timely Deliveries
```sql
-- Query to assess supplier reliability based on delivery times
WITH supplier_performance AS (
    SELECT 
        Ship_Mode,
        AVG(DATEDIFF(NOW(), Order_Date)) AS avg_delivery_days,
        COUNT(CASE WHEN Quantity = Sales THEN 1 END) / COUNT(*) AS on_time_delivery_rate
    FROM inventory.inventory_data_1
    GROUP BY Ship_Mode
)
SELECT 
    Ship_Mode,
    avg_delivery_days,
    on_time_delivery_rate * 100 AS on_time_delivery_percentage
FROM supplier_performance
ORDER BY on_time_delivery_percentage ASC;
```
## Inventory Cost Analysis
### Inventory Carrying Costs
```sql
WITH inventory_cost AS (
    SELECT 
        Product_ID, 
        Product_Name, 
        SUM(Quantity * cost) AS total_inventory_cost,
        SUM(Quantity) AS total_units
    FROM inventory.inventory_data_1
    GROUP BY Product_ID, Product_Name
)
SELECT 
    Product_ID,
    Product_Name,
    total_inventory_cost,
    total_units,
    total_inventory_cost / total_units AS carrying_cost_per_unit
FROM inventory_cost
ORDER BY total_inventory_cost DESC;
```
## Customer Demand Analysis
### Customer Purchasing behavior
```sql
-- Query to segment customers based on purchasing behavior
WITH customer_segments AS (
    SELECT 
        Customer_ID,
        SUM(Sales * Quantity) AS total_spent,
        COUNT(DISTINCT Order_Date) AS Order_frequency,
        MAX(Order_Date) AS last_order_date
    FROM inventory.inventory_data_1
    GROUP BY Customer_ID
),
customer_classification AS (
    SELECT 
        Customer_ID,
        total_spent,
        Order_frequency,
        CASE 
            WHEN total_spent > 1000 THEN 'High Value'
            WHEN total_spent BETWEEN 500 AND 1000 THEN 'Medium Value'
            ELSE 'Low Value'
        END AS customer_segment,
        CASE 
            WHEN DATEDIFF(NOW(), last_order_date) <= 30 THEN 'Active'
            ELSE 'Inactive'
        END AS activity_status
    FROM customer_segments
)
SELECT 
    Customer_ID,
    total_spent,
    Order_frequency,
    customer_segment,
    activity_status
FROM customer_classification
ORDER BY total_spent DESC;
```

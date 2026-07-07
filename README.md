# Target--SQL-Business-Case
# 🎯 Target Brazil — E-Commerce SQL Business Case

SQL analysis of Target's Brazil marketplace operations: ~100,000 orders placed between
September 2016 and October 2018, covering order trends, customer geography, order
economics (price, freight, payments), and delivery performance.

## 📌 Business Problem

Analyse Target's Brazil e-commerce dataset to extract insights on order growth,
customer distribution, pricing/freight economics, delivery performance, and payment
behaviour — and turn those into actionable recommendations.

## 🗂️ Files

```
target-sql-project/
├── sql/
│   └── target_brazil_analysis.sql     # All 19 queries, organized by section, tested against real data
├── reports/
│   ├── Target_Brazil_SQL_Report.docx  # Submission-ready report: query + output table + insights
│   └── Target_Brazil_SQL_Report.pdf   # Same report, PDF (8 pages)
└── README.md
```

## 🗄️ Dataset

8 CSV files (not included here — see your original upload):

| File | Description |
|---|---|
| `customers.csv` | Customer ID, location (city/state/zip) |
| `sellers.csv` | Seller ID, location |
| `orders.csv` | Order status and purchase/delivery timestamps |
| `order_items.csv` | Line-item price and freight value per order |
| `payments.csv` | Payment type, installments, amount |
| `order_reviews.csv` | Review scores and comments |
| `products.csv` | Product category and dimensions |
| `geolocation.csv` | Zip-code-level lat/long |

## 🔍 What the SQL Covers

1. **Initial Exploration** — column data types, order date range, customer city/state counts
2. **In-Depth Exploration** — year-on-year order growth, monthly seasonality, time-of-day ordering pattern
3. **Evolution of E-Commerce** — month-on-month orders by state, customer distribution by state
4. **Impact on Economy** — YoY % increase in order value, total/average order price and freight by state
5. **Sales, Freight & Delivery Time** — delivery time vs. estimate, top/bottom 5 states by freight, delivery time, and delivery-vs-estimate gap
6. **Payments** — month-on-month orders by payment type, orders by installment count

## 📊 Key Findings

- Order value grew **~137%** from 2017 to 2018 (Jan–Aug comparison)
- **São Paulo (SP)** dominates: 40,302 customers, lowest average freight (R$15.15), fastest delivery (8.8 days)
- Remote states (RR, AP, AM, AC) have the longest delivery times **and** the widest gap versus their estimated delivery date — Target appears to pad delivery estimates generously for these regions
- **Credit card** is the dominant payment method every month; single-payment orders are most common, with 10-installment EMI plans the most popular financing option
- Order volume peaks **May–August** and dips in **September–October**; **Afternoon** is the most popular ordering window

Full write-up of insights and recommendations is in
[`reports/Target_Brazil_SQL_Report.docx`](reports/Target_Brazil_SQL_Report.docx).

## 🛠️ How to Run

The queries are written in **MySQL syntax** (`YEAR()`, `MONTH()`, `HOUR()`, `DATEDIFF()`).

1. Create a database and import the 8 CSVs as tables (matching the file names as table names — `customers`, `sellers`, `order_items`, `geolocation`, `payments`, `order_reviews`, `orders`, `products`).
2. Open `sql/target_brazil_analysis.sql` in MySQL Workbench (or any MySQL-compatible client).
3. Run section by section — each section is clearly commented and matches the assignment's rubric structure.

**Note for other SQL dialects:**
| MySQL | PostgreSQL | SQLite |
|---|---|---|
| `YEAR(col)` | `EXTRACT(YEAR FROM col)` | `strftime('%Y', col)` |
| `MONTH(col)` | `EXTRACT(MONTH FROM col)` | `strftime('%m', col)` |
| `HOUR(col)` | `EXTRACT(HOUR FROM col)` | `strftime('%H', col)` |
| `DATEDIFF(a, b)` | `a::date - b::date` | `julianday(a) - julianday(b)` |

## 👤 Author

**Krishna Jinde** — Business Analyst | MSc Business Analytics, Swansea University
[LinkedIn](https://linkedin.com/in/krishna-jinde-70845285)

## 📜 Full SQL Script

```sql
/* =====================================================================
   TARGET BRAZIL — E-COMMERCE SQL BUSINESS CASE
   Tables: customers, sellers, order_items, geolocation, payments,
           order_reviews, orders, products
   Dialect: MySQL (tested for logical correctness against the real
   dataset; adjust date functions if you're running this on
   PostgreSQL / SQL Server / SQLite — notes given where relevant)
   ===================================================================== */


/* =====================================================================
   SECTION 1: INITIAL EXPLORATION
   ===================================================================== */

-- 1.1 Data type of all columns in the "customers" table
DESCRIBE customers;
-- (Equivalent in PostgreSQL: use information_schema.columns;
--  in SQLite: PRAGMA table_info(customers);)


-- 1.2 Time range between which the orders were placed
SELECT
    MIN(order_purchase_timestamp) AS first_order_date,
    MAX(order_purchase_timestamp) AS last_order_date
FROM orders;
-- Result: 2016-09-04 21:15:19  to  2018-10-17 17:30:18


-- 1.3 Count of Cities & States of customers who ordered during the period
SELECT
    COUNT(DISTINCT customer_city)  AS total_cities,
    COUNT(DISTINCT customer_state) AS total_states
FROM customers;
-- Result: 4,119 cities across 27 states



/* =====================================================================
   SECTION 2: IN-DEPTH EXPLORATION
   ===================================================================== */

-- 2.1 Is there a growing trend in the number of orders placed over the years?
SELECT
    YEAR(order_purchase_timestamp)  AS order_year,
    COUNT(order_id)                 AS total_orders
FROM orders
GROUP BY order_year
ORDER BY order_year;
-- Result: 2016 = 329 (partial year), 2017 = 45,101, 2018 = 54,011 (through Oct)
-- => Clear year-on-year growth


-- 2.2 Monthly seasonality in number of orders placed (across all years combined)
SELECT
    MONTH(order_purchase_timestamp) AS order_month,
    COUNT(order_id)                 AS total_orders
FROM orders
GROUP BY order_month
ORDER BY order_month;
-- Result: orders peak May-August, dip in Sept-Oct, and again in Dec


-- 2.3 Time of day Brazilian customers mostly place orders
-- 0-6 Dawn | 7-12 Morning | 13-18 Afternoon | 19-23 Night
SELECT
    CASE
        WHEN HOUR(order_purchase_timestamp) BETWEEN 0  AND 6  THEN 'Dawn'
        WHEN HOUR(order_purchase_timestamp) BETWEEN 7  AND 12 THEN 'Morning'
        WHEN HOUR(order_purchase_timestamp) BETWEEN 13 AND 18 THEN 'Afternoon'
        ELSE 'Night'
    END AS time_of_day,
    COUNT(order_id) AS total_orders
FROM orders
GROUP BY time_of_day
ORDER BY total_orders DESC;
-- Result: Afternoon (38,135) > Night (28,331) > Morning (27,733) > Dawn (5,242)



/* =====================================================================
   SECTION 3: EVOLUTION OF E-COMMERCE ORDERS IN BRAZIL
   ===================================================================== */

-- 3.1 Month-on-month number of orders placed in each state
SELECT
    c.customer_state                 AS state,
    YEAR(o.order_purchase_timestamp)  AS order_year,
    MONTH(o.order_purchase_timestamp) AS order_month,
    COUNT(o.order_id)                 AS total_orders
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
GROUP BY state, order_year, order_month
ORDER BY state, order_year, order_month;


-- 3.2 How are customers distributed across all the states?
SELECT
    customer_state               AS state,
    COUNT(DISTINCT customer_unique_id) AS total_customers
FROM customers
GROUP BY state
ORDER BY total_customers DESC;
-- Result: SP (40,302) dominates, followed by RJ (12,384), MG (11,259)



/* =====================================================================
   SECTION 4: IMPACT ON ECONOMY
   ===================================================================== */

-- 4.1 % increase in cost of orders from 2017 to 2018 (Jan-Aug only)
-- Using payment_value from the payments table
SELECT
    yr_2017.total_2017 AS total_cost_2017_jan_aug,
    yr_2018.total_2018 AS total_cost_2018_jan_aug,
    ROUND((yr_2018.total_2018 - yr_2017.total_2017) * 100.0 / yr_2017.total_2017, 2)
        AS pct_increase
FROM
    (SELECT SUM(p.payment_value) AS total_2017
     FROM orders o
     JOIN payments p ON o.order_id = p.order_id
     WHERE YEAR(o.order_purchase_timestamp) = 2017
       AND MONTH(o.order_purchase_timestamp) BETWEEN 1 AND 8) yr_2017,
    (SELECT SUM(p.payment_value) AS total_2018
     FROM orders o
     JOIN payments p ON o.order_id = p.order_id
     WHERE YEAR(o.order_purchase_timestamp) = 2018
       AND MONTH(o.order_purchase_timestamp) BETWEEN 1 AND 8) yr_2018;
-- Result: 2017 (Jan-Aug) = R$3,669,022.12 | 2018 (Jan-Aug) = R$8,694,733.84
-- => ~136.98% increase in order value year-on-year


-- 4.2 Total & Average order price by state
SELECT
    c.customer_state              AS state,
    ROUND(SUM(oi.price), 2)       AS total_order_price,
    ROUND(AVG(oi.price), 2)       AS avg_order_price
FROM order_items oi
JOIN orders o     ON oi.order_id = o.order_id
JOIN customers c  ON o.customer_id = c.customer_id
GROUP BY state
ORDER BY total_order_price DESC;
-- Result: SP leads with R$5.2M total; BA has highest avg order price (R$134.60)


-- 4.3 Total & Average freight value by state
SELECT
    c.customer_state                   AS state,
    ROUND(SUM(oi.freight_value), 2)    AS total_freight_value,
    ROUND(AVG(oi.freight_value), 2)    AS avg_freight_value
FROM order_items oi
JOIN orders o     ON oi.order_id = o.order_id
JOIN customers c  ON o.customer_id = c.customer_id
GROUP BY state
ORDER BY total_freight_value DESC;
-- Result: SP has lowest avg freight (R$15.15) despite highest volume;
--         PE has highest avg freight (R$32.92) among top-volume states



/* =====================================================================
   SECTION 5: ANALYSIS BASED ON SALES, FREIGHT & DELIVERY TIME
   ===================================================================== */

-- 5.1 Delivery time & difference from estimated delivery date — single query
SELECT
    order_id,
    order_purchase_timestamp,
    order_delivered_customer_date,
    order_estimated_delivery_date,
    DATEDIFF(order_delivered_customer_date, order_purchase_timestamp)
        AS time_to_deliver_days,
    DATEDIFF(order_delivered_customer_date, order_estimated_delivery_date)
        AS diff_estimated_delivery_days
FROM orders
WHERE order_delivered_customer_date IS NOT NULL;
-- Negative diff_estimated_delivery_days = delivered EARLIER than estimated


-- 5.2 Top 5 states — HIGHEST average freight value
SELECT
    c.customer_state            AS state,
    ROUND(AVG(oi.freight_value), 2) AS avg_freight_value
FROM order_items oi
JOIN orders o     ON oi.order_id = o.order_id
JOIN customers c  ON o.customer_id = c.customer_id
GROUP BY state
ORDER BY avg_freight_value DESC
LIMIT 5;
-- Result: RR (42.98), PB (42.72), RO (41.07), AC (40.07), PI (39.15)

-- 5.2 Top 5 states — LOWEST average freight value
SELECT
    c.customer_state            AS state,
    ROUND(AVG(oi.freight_value), 2) AS avg_freight_value
FROM order_items oi
JOIN orders o     ON oi.order_id = o.order_id
JOIN customers c  ON o.customer_id = c.customer_id
GROUP BY state
ORDER BY avg_freight_value ASC
LIMIT 5;
-- Result: SP (15.15), PR (20.53), MG (20.63), RJ (20.96), DF (21.04)


-- 5.3 Top 5 states — HIGHEST average delivery time
SELECT
    c.customer_state AS state,
    ROUND(AVG(DATEDIFF(o.order_delivered_customer_date, o.order_purchase_timestamp)), 2)
        AS avg_delivery_days
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_delivered_customer_date IS NOT NULL
GROUP BY state
ORDER BY avg_delivery_days DESC
LIMIT 5;
-- Result: RR (29.39), AP (27.19), AM (26.43), AL (24.54), PA (23.77)

-- 5.3 Top 5 states — LOWEST average delivery time
SELECT
    c.customer_state AS state,
    ROUND(AVG(DATEDIFF(o.order_delivered_customer_date, o.order_purchase_timestamp)), 2)
        AS avg_delivery_days
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_delivered_customer_date IS NOT NULL
GROUP BY state
ORDER BY avg_delivery_days ASC
LIMIT 5;
-- Result: SP (8.76), PR (11.99), MG (12.01), DF (12.97), SC (14.96)


-- 5.4 Top 5 states where delivery is fastest compared to the estimate
-- (most negative avg_diff_days = delivered furthest ahead of schedule)
SELECT
    c.customer_state AS state,
    ROUND(AVG(DATEDIFF(o.order_delivered_customer_date, o.order_estimated_delivery_date)), 2)
        AS avg_diff_days
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_delivered_customer_date IS NOT NULL
GROUP BY state
ORDER BY avg_diff_days ASC
LIMIT 5;
-- Result: AC (-20.08), RO (-19.40), AP (-19.06), AM (-18.85), RR (-16.59)
-- Note: these are also the states with the highest raw delivery times (5.3) —
-- Target sets generous delivery estimates for remote states, so actual
-- performance still beats the (padded) estimate.



/* =====================================================================
   SECTION 6: ANALYSIS BASED ON PAYMENTS
   ===================================================================== */

-- 6.1 Month-on-month number of orders placed, by payment type
SELECT
    YEAR(o.order_purchase_timestamp)  AS order_year,
    MONTH(o.order_purchase_timestamp) AS order_month,
    p.payment_type,
    COUNT(DISTINCT o.order_id)        AS total_orders
FROM orders o
JOIN payments p ON o.order_id = p.order_id
GROUP BY order_year, order_month, p.payment_type
ORDER BY order_year, order_month, total_orders DESC;


-- 6.2 Number of orders placed by payment installment count
SELECT
    payment_installments,
    COUNT(DISTINCT order_id) AS total_orders
FROM payments
GROUP BY payment_installments
ORDER BY payment_installments;
-- Result: single (1) installment dominates with 49,060 orders;
--         10-installment plans are the most popular EMI option (5,315 orders)
```

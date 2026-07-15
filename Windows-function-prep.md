# SQL Window Functions Interview Guide

## What are Window Functions?

A window function performs calculations across a set of related rows
**without reducing the number of rows returned**.

**Syntax**

``` sql
FUNCTION_NAME() OVER (
    PARTITION BY column_name
    ORDER BY column_name
)
```


## Common Window Functions

  Function           Purpose                 Typical Use
  ------------------ ----------------------- ------------------------------------
  ROW_NUMBER()       Unique sequence         Pagination, remove duplicates
  RANK()             Ranking with gaps       Leaderboards
  DENSE_RANK()       Ranking without gaps    Top-N queries
  LEAD()             Read the next row       Compare current vs next record
  LAG()              Read the previous row   Compare current vs previous record
  FIRST_VALUE()      First value in window   Baseline comparisons
  LAST_VALUE()       Last value in window    Final value in partition
  NTILE(n)           Divide into buckets     Quartiles/percentiles
  SUM() OVER         Running total           Financial reports
  AVG() OVER         Moving average          Sales trends
  MAX()/MIN() OVER   Max/Min in partition    Highest/Lowest salary

## GROUP BY vs Window Functions

  GROUP BY            Window Function
  ------------------- ----------------------------------
  Collapses rows      Preserves rows
  One row per group   All rows remain
  Aggregate only      Aggregate + Ranking + Comparison

------------------------------------------------------------------------

# LEAD() and LAG()

These are among the most practical window functions because they let you
compare rows **without self-joins**.

## LEAD()

Returns a value from a future row.

``` sql
LEAD(column_name, offset, default_value)
OVER (ORDER BY column_name)
```

Example:

``` sql
SELECT
    month,
    sales,
    LEAD(sales) OVER (ORDER BY month) AS next_month_sales
FROM sales;
```

  Month     Sales   Next Month Sales
  ------- ------- ------------------
  Jan         100                150
  Feb         150                170
  Mar         170               NULL

### Real Interview Scenarios for LEAD()

1.  Compare current month's sales with next month.
2.  Find employees whose next promotion date is within 30 days.
3.  Detect the next flight or shipment for a customer.
4.  Compare current stock price with the next trading day.
5.  Identify the next order placed by the same customer.

Example:

``` sql
SELECT
    order_date,
    amount,
    LEAD(amount) OVER (ORDER BY order_date) AS next_order_amount
FROM orders;
```

------------------------------------------------------------------------

## LAG()

Returns a value from a previous row.

``` sql
LAG(column_name, offset, default_value)
OVER (ORDER BY column_name)
```

Example:

``` sql
SELECT
    month,
    sales,
    LAG(sales) OVER (ORDER BY month) AS previous_month_sales
FROM sales;
```

  Month     Sales   Previous Month Sales
  ------- ------- ----------------------
  Jan         100                   NULL
  Feb         150                    100
  Mar         170                    150

### Real Interview Scenarios for LAG()

1.  Month-over-month sales comparison.
2.  Compare today's API latency with yesterday's latency.
3.  Detect salary changes.
4.  Find gaps between login events.
5.  Calculate daily stock price changes.

Example:

``` sql
SELECT
    log_date,
    response_time,
    response_time - LAG(response_time)
        OVER (ORDER BY log_date) AS latency_difference
FROM api_metrics;
```

------------------------------------------------------------------------

## LEAD vs LAG

  LEAD                   LAG
  ---------------------- ------------------------
  Looks forward          Looks backward
  Next row               Previous row
  Forecast comparisons   Historical comparisons

------------------------------------------------------------------------

## Frequently Asked Interview Questions

### Difference between ROW_NUMBER(), RANK() and DENSE_RANK()

-   ROW_NUMBER(): Always unique.
-   RANK(): Same rank for ties; gaps appear.
-   DENSE_RANK(): Same rank for ties; no gaps.

### Why use LEAD/LAG instead of Self Join?

-   Simpler SQL
-   Easier to read
-   Better performance in many cases
-   Ideal for time-series and sequential comparisons

### Can Window Functions be used in WHERE?

No. Use a CTE or subquery.

``` sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT *
FROM ranked
WHERE rn = 1;
```

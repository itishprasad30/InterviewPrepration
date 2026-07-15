# SQL Interview Prep — 50+ Questions (Basic to Advanced + Window Functions)

Format: **What / Why / How** + example. Examples use an `employees` / `payroll` style schema close to your Humine domain.

```sql
-- Reference schema used in examples
employees(id, name, dept_id, salary, manager_id, hire_date)
departments(id, name)
payroll(id, employee_id, month, amount, status)
```

---

## Section 1: SQL Basics

**1. What is SQL and what are its sub-languages?**
- **DDL** (Data Definition) — `CREATE`, `ALTER`, `DROP` — defines structure.
- **DML** (Data Manipulation) — `SELECT`, `INSERT`, `UPDATE`, `DELETE` — works with data.
- **DCL** (Data Control) — `GRANT`, `REVOKE` — permissions.
- **TCL** (Transaction Control) — `COMMIT`, `ROLLBACK`, `SAVEPOINT`.

**2. `WHERE` vs `HAVING`?**
- `WHERE` filters rows **before** grouping; `HAVING` filters groups **after** `GROUP BY`/aggregation.
```sql
SELECT dept_id, AVG(salary) 
FROM employees 
WHERE hire_date > '2022-01-01'   -- row filter first
GROUP BY dept_id
HAVING AVG(salary) > 60000;      -- group filter after
```

**3. `DELETE` vs `TRUNCATE` vs `DROP`?**
- `DELETE` — removes rows (can use `WHERE`), logged, rollback-able, triggers fire.
- `TRUNCATE` — removes all rows instantly, minimal logging, resets identity, can't use `WHERE`, generally can't rollback (DB-dependent).
- `DROP` — removes the entire table structure + data.

**4. `UNION` vs `UNION ALL`?**
- `UNION` removes duplicate rows (slower, does a sort/dedupe). `UNION ALL` keeps everything (faster). Use `UNION ALL` unless you specifically need deduplication.

**5. What is a Primary Key vs Unique Key?**
- Primary key — uniquely identifies each row, only one per table, can't be `NULL`.
- Unique key — also enforces uniqueness, but a table can have multiple, and it **can** allow one `NULL` (DB-dependent).

**6. What is a Foreign Key?**
- **What:** A column referencing the primary key of another table.
- **Why:** Enforces referential integrity — you can't insert a `payroll.employee_id` that doesn't exist in `employees`.
```sql
ALTER TABLE payroll ADD CONSTRAINT fk_emp 
FOREIGN KEY (employee_id) REFERENCES employees(id);
```

**7. What are the types of SQL JOINs?**
- `INNER JOIN` — only matching rows in both tables.
- `LEFT JOIN` — all rows from left + matches from right (NULL if no match).
- `RIGHT JOIN` — all rows from right + matches from left.
- `FULL OUTER JOIN` — all rows from both, matched where possible.
- `CROSS JOIN` — Cartesian product (every row × every row).
```sql
SELECT e.name, d.name AS dept
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id;
```

**8. Self Join — what and when?**
- **What:** A table joined with itself.
- **Why:** Useful for hierarchical data — e.g., finding each employee's manager (who is also in the `employees` table).
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

**9. What is Normalization? Name the normal forms briefly.**
- **What:** Organizing tables to reduce redundancy and avoid update anomalies.
- **1NF** — atomic columns, no repeating groups.
- **2NF** — 1NF + no partial dependency on part of a composite key.
- **3NF** — 2NF + no transitive dependency (non-key columns depend only on the key).
- **Why:** Prevents data duplication/inconsistency, though sometimes you deliberately denormalize for read performance.

**10. What is `NULL` and how is it different from 0 or empty string?**
- `NULL` means "unknown/absent," not zero or empty. Any arithmetic/comparison with `NULL` yields `NULL` (not true/false) — use `IS NULL` / `IS NOT NULL`, not `= NULL`.

---

## Section 2: Aggregate Functions & Grouping

**11. What are aggregate functions?**
`COUNT()`, `SUM()`, `AVG()`, `MIN()`, `MAX()` — operate on a set of rows, return a single value.

**12. `COUNT(*)` vs `COUNT(column)` vs `COUNT(DISTINCT column)`?**
- `COUNT(*)` — counts all rows including NULLs.
- `COUNT(column)` — counts non-NULL values in that column.
- `COUNT(DISTINCT column)` — counts unique non-NULL values.

**13. `GROUP BY` — how does it work with multiple columns?**
- Groups rows sharing the same combination of values across all listed columns.
```sql
SELECT dept_id, status, COUNT(*) 
FROM payroll p JOIN employees e ON p.employee_id = e.id
GROUP BY dept_id, status;
```

**14. Can you use a column in `SELECT` that's not in `GROUP BY`?**
- Only if it's wrapped in an aggregate function, or it's functionally dependent on the grouped column (MySQL is lenient here; standard SQL/Postgres enforce it strictly).

---

## Section 3: Subqueries & Set Operations

**15. What is a subquery? Correlated vs non-correlated?**
- **Non-correlated** — runs independently, once. **Correlated** — references the outer query, runs once per outer row (like a loop, can be slower).
```sql
-- Non-correlated
SELECT * FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);

-- Correlated: employees earning more than their dept's average
SELECT * FROM employees e1
WHERE salary > (SELECT AVG(salary) FROM employees e2 WHERE e2.dept_id = e1.dept_id);
```

**16. `IN` vs `EXISTS`?**
- `IN` — checks value against a list/subquery result set; can be slow on large subqueries (though optimizers often rewrite it).
- `EXISTS` — checks if a subquery returns any row at all, stops at first match — often faster for large correlated checks.

**17. What is a CTE (Common Table Expression)?**
- **What:** A named temporary result set defined with `WITH`, used within a single query.
- **Why:** Improves readability over nested subqueries; can be recursive.
```sql
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 80000
)
SELECT dept_id, COUNT(*) FROM high_earners GROUP BY dept_id;
```

**18. Recursive CTE — example?**
- Useful for hierarchical data, e.g., org chart traversal.
```sql
WITH RECURSIVE org_chart AS (
    SELECT id, name, manager_id, 1 AS level FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart;
```

**19. What is a derived table (inline view)?**
A subquery used in the `FROM` clause, treated as a temporary table for that query.
```sql
SELECT dept_id, avg_sal FROM (
    SELECT dept_id, AVG(salary) AS avg_sal FROM employees GROUP BY dept_id
) t WHERE avg_sal > 50000;
```

---

## Section 4: Indexes & Performance

**20. What is an Index and why use one?**
- **What:** A separate data structure (usually B-tree) that speeds up lookups on a column, at the cost of extra storage and slower writes (index must be updated too).
- **Why:** Without an index, the DB does a full table scan for lookups — O(n). With an index, it's roughly O(log n).

**21. Clustered vs Non-clustered index?**
- **Clustered** — determines the physical order of data in the table (only one per table, usually the primary key).
- **Non-clustered** — a separate structure pointing back to the actual row (a table can have many).

**22. When would an index NOT help / hurt?**
- Low-cardinality columns (e.g., a `status` column with only 3 values), tables with heavy writes (index maintenance overhead), or when the query uses a function on the indexed column (`WHERE UPPER(name) = 'X'` won't use a plain index on `name`).

**23. What is `EXPLAIN` / execution plan used for?**
- Shows how the DB engine will execute a query (index usage, join type, scan type) — key tool for diagnosing slow queries.

**24. What is Query Optimization — a few practical tips?**
- Select only needed columns (avoid `SELECT *`), index columns used in `WHERE`/`JOIN`/`ORDER BY`, avoid functions on indexed columns in `WHERE`, use `LIMIT` where possible, avoid N+1 query patterns from the app layer (relevant to your Ebean/JPA work).

---

## Section 5: Transactions & Constraints

**25. What is a Transaction? ACID properties?**
- **Atomicity** — all or nothing.
- **Consistency** — DB moves from one valid state to another.
- **Isolation** — concurrent transactions don't interfere.
- **Durability** — once committed, changes survive crashes.

**26. What are Transaction Isolation Levels?**
- `READ UNCOMMITTED` — can see uncommitted changes from other transactions (dirty reads).
- `READ COMMITTED` — only sees committed data (default in many DBs).
- `REPEATABLE READ` — same query returns same result within a transaction.
- `SERIALIZABLE` — strictest, transactions behave as if executed one at a time.

**27. What is a Dirty Read, Non-repeatable Read, Phantom Read?**
- **Dirty read** — reading uncommitted data from another transaction.
- **Non-repeatable read** — re-reading a row gives different values because another transaction updated/committed it in between.
- **Phantom read** — re-running a query returns *different rows* because another transaction inserted/deleted matching rows.

**28. What are constraints? Name common types.**
`NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY`, `CHECK`, `DEFAULT` — enforce data integrity rules at the DB level.

**29. What is a Deadlock in SQL and how do DBs handle it?**
Two transactions each holding a lock the other needs. Most DBs detect this and automatically abort one transaction (deadlock victim) to break the cycle.

---

## Section 6: Window Functions (Advanced — High-Value for Interviews)

**30. What is a Window Function and how is it different from `GROUP BY`?**
- **What:** Performs a calculation across a set of rows related to the current row, **without collapsing rows into groups** — unlike `GROUP BY`/aggregates, you still get one row per input row.
- **Why:** Lets you compute running totals, rankings, or comparisons to other rows while keeping row-level detail.
- **How (syntax):**
```sql
function_name(...) OVER (
    [PARTITION BY column]   -- like GROUP BY, but doesn't collapse rows
    [ORDER BY column]       -- defines the order for ranking/running calcs
    [ROWS/RANGE frame]      -- optional: defines the "window" of rows
)
```

**31. `ROW_NUMBER()` — what and example?**
- Assigns a unique sequential number to each row within a partition, no ties.
```sql
SELECT name, dept_id, salary,
       ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
FROM employees;
-- Use case: find each department's top earner -> WHERE rn = 1
```

**32. `RANK()` vs `DENSE_RANK()` vs `ROW_NUMBER()`?**
- `ROW_NUMBER()` — always unique, no gaps, no ties (1,2,3,4).
- `RANK()` — ties get the same rank, but leaves a gap after (1,2,2,4).
- `DENSE_RANK()` — ties get the same rank, no gap after (1,2,2,3).
```sql
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS rnk,
       DENSE_RANK() OVER (ORDER BY salary DESC) AS drnk
FROM employees;
```

**33. How do you find the Nth highest salary using window functions?**
```sql
SELECT * FROM (
    SELECT name, salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) t WHERE rnk = 3;   -- 3rd highest salary
```

**34. `LAG()` and `LEAD()` — what and example?**
- **What:** Access a value from a previous (`LAG`) or next (`LEAD`) row within the same ordered partition, without a self-join.
- **Why:** Great for comparing a row to the "previous month" or "next event" — e.g., month-over-month payroll comparison.
```sql
SELECT employee_id, month, amount,
       LAG(amount) OVER (PARTITION BY employee_id ORDER BY month) AS prev_month_amount,
       amount - LAG(amount) OVER (PARTITION BY employee_id ORDER BY month) AS change
FROM payroll;
```

**35. Running total (cumulative sum) — how?**
```sql
SELECT employee_id, month, amount,
       SUM(amount) OVER (PARTITION BY employee_id ORDER BY month) AS running_total
FROM payroll;
```

**36. Moving average — how with window frames?**
```sql
SELECT employee_id, month, amount,
       AVG(amount) OVER (
           PARTITION BY employee_id 
           ORDER BY month 
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) AS moving_avg_3month
FROM payroll;
```

**37. What is `ROWS` vs `RANGE` in a window frame?**
- `ROWS` — counts physical rows before/after (e.g., "2 preceding rows").
- `RANGE` — counts based on logical value ranges of the `ORDER BY` column (e.g., same date can span multiple rows treated as one range).

**38. `NTILE(n)` — what and use case?**
- Divides rows within a partition into `n` roughly equal buckets — e.g., splitting employees into salary quartiles.
```sql
SELECT name, salary, NTILE(4) OVER (ORDER BY salary DESC) AS quartile FROM employees;
```

**39. `FIRST_VALUE()` / `LAST_VALUE()` — what?**
Returns the first/last value in the ordered window frame — e.g., each employee's first-ever payroll amount alongside every row.
```sql
SELECT employee_id, month, amount,
       FIRST_VALUE(amount) OVER (PARTITION BY employee_id ORDER BY month) AS first_payroll
FROM payroll;
```

**40. Can you use a window function in `WHERE`?**
- **No** — window functions are evaluated after `WHERE`/`GROUP BY`/`HAVING`, so you must wrap the query and filter in an outer query (as shown in Q33).

**41. Real interview problem: find employees earning above their department average, using a window function (no correlated subquery).**
```sql
SELECT * FROM (
    SELECT name, dept_id, salary,
           AVG(salary) OVER (PARTITION BY dept_id) AS dept_avg
    FROM employees
) t
WHERE salary > dept_avg;
```

---

## Section 7: More Advanced / Practical

**42. What is a View?**
A virtual table defined by a stored query — doesn't store data itself (unless materialized), simplifies complex/repeated queries and can restrict column access.

**43. Materialized View vs regular View?**
Materialized view physically stores the query result and needs periodic refresh — trades staleness for read speed. Regular view always runs the underlying query live.

**44. What is a Stored Procedure?**
Precompiled SQL logic stored in the DB, callable by name — reduces network round-trips, centralizes business logic, but can make version control/testing harder (many teams now prefer keeping logic in the application layer, as in your Spring Boot/Ebean setup).

**45. What is a Trigger?**
Code that automatically runs in response to an `INSERT`/`UPDATE`/`DELETE` on a table — e.g., auto-logging changes to an audit table.

**46. Composite index — what and column order matters why?**
An index on multiple columns. Order matters because the index is only efficiently usable for queries that filter on a **leading prefix** of those columns (index on `(dept_id, salary)` helps `WHERE dept_id = X`, and `WHERE dept_id = X AND salary > Y`, but not `WHERE salary > Y` alone).

**47. What is Sharding vs Partitioning?**
- **Partitioning** — splitting a large table into smaller pieces *within the same DB* (by range, hash, list) for manageability/performance.
- **Sharding** — splitting data *across multiple separate DB instances/servers*, typically for horizontal scaling.

**48. `INNER JOIN` vs subquery with `IN` — which is generally faster?**
Depends on the optimizer and data size, but `JOIN`s are often better optimized for large datasets since the engine can pick the best join strategy (hash join, merge join), whereas correlated subqueries can force row-by-row evaluation.

**49. How do you find duplicate rows in a table?**
```sql
SELECT email, COUNT(*) 
FROM employees 
GROUP BY email 
HAVING COUNT(*) > 1;
```

**50. How do you delete duplicate rows, keeping only one?**
```sql
DELETE FROM employees
WHERE id NOT IN (
    SELECT MIN(id) FROM employees GROUP BY email
);
-- Or with window functions:
DELETE FROM employees
WHERE id IN (
    SELECT id FROM (
        SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
        FROM employees
    ) t WHERE rn > 1
);
```

**51. Bonus — Pagination with `LIMIT`/`OFFSET`, and why it's slow for deep pages?**
```sql
SELECT * FROM employees ORDER BY id LIMIT 20 OFFSET 1000;
```
Slow for large offsets because the DB still scans/skips all the preceding rows. **Keyset pagination** (`WHERE id > last_seen_id ORDER BY id LIMIT 20`) is more efficient for deep pagination.

---

## Quick tips for your interviews
- **Window functions are a strong differentiator at 3-5 YOE** — interviewers love asking Nth-highest-salary and running-total problems specifically because so many candidates only know `GROUP BY`. Make sure Q31-41 are second nature.
- Tie index/query optimization answers to your Ebean/MySQL work on Humine — real N+1 or slow-query war stories land very well.
- Be ready to write these live on a whiteboard/shared doc, not just explain them — practice typing them out without an IDE's autocomplete.

Want a mock round mixing SQL, Java, and Spring Boot questions together like a real panel interview would?

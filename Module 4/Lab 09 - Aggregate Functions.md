# 🧩 **Lab 09 - Aggregate Functions, GROUP BY, and HAVING (PostgreSQL Hands-on Lab)**

---

## 🧭 **Introduction**

Aggregate functions are the foundation of analytics in SQL: they summarize many rows into meaningful metrics (counts, totals, averages). In PostgreSQL, understanding how aggregates interact with **NULLs**, **GROUP BY**, and **HAVING** is key to producing correct reports.

This lab is designed to teach the most common “on the job” patterns:

- `COUNT(*)` vs `COUNT(column)` (and why NULL matters)
- Grouping with `GROUP BY`
- Filtering **before** vs **after** aggregation (`WHERE` vs `HAVING`)
- Aggregating across joins while keeping “zero” groups using OUTER JOIN + `COALESCE`

---

## 🎯 **Objectives**

By the end of this lab, learners will be able to:

- Use aggregate functions: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`
- Write correct `GROUP BY` queries (including multi-column grouping)
- Filter grouped results using `HAVING`
- Explain and avoid common aggregate mistakes (NULL handling, join duplication, `WHERE` vs `HAVING`)

---

## 🧠 **What are aggregate functions?**

Aggregate functions compute a single result from multiple input rows (either the whole table or per group).

- Aggregates typically **ignore NULL values** (example: `AVG(col)` ignores NULLs)
- `COUNT(*)` counts **rows**, while `COUNT(col)` counts **non-NULL values**

---

## 📊 **Common aggregate functions**


| Function     | Meaning                   | Notes                                 |
| ------------ | ------------------------- | ------------------------------------- |
| `COUNT(*)`   | number of rows            | counts NULL rows too (it counts rows) |
| `COUNT(col)` | number of non-NULL values | ignores NULL values in `col`          |
| `SUM(col)`   | total                     | ignores NULL values                   |
| `AVG(col)`   | average                   | ignores NULL values                   |
| `MIN(col)`   | smallest                  | ignores NULL values                   |
| `MAX(col)`   | largest                   | ignores NULL values                   |


---

## 🧰 **Step 0 – Setup**

This lab creates its own schema (`lab09`) so it won’t conflict with previous/next labs.

Run once:

```sql
BEGIN;

DROP SCHEMA IF EXISTS lab09 CASCADE;
CREATE SCHEMA lab09;

CREATE TABLE lab09.department (
  dept_id   int PRIMARY KEY,
  dept_name text NOT NULL UNIQUE
);

CREATE TABLE lab09.employee (
  emp_id     int PRIMARY KEY,
  first_name text NOT NULL,
  dept_id    int NULL REFERENCES lab09.department(dept_id)
);

-- Salary is optional: some employees will have no salary row, and one will have a NULL salary
CREATE TABLE lab09.salary (
  emp_id         int PRIMARY KEY REFERENCES lab09.employee(emp_id),
  monthly_salary numeric(12,2) NULL CHECK (monthly_salary IS NULL OR monthly_salary > 0)
);

-- Simple “fact” table to practice SUM/AVG by category and date
CREATE TABLE lab09.sales (
  sale_id    int GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  dept_id    int NOT NULL REFERENCES lab09.department(dept_id),
  sale_month date NOT NULL,
  amount     numeric(12,2) NOT NULL CHECK (amount >= 0)
);

INSERT INTO lab09.department (dept_id, dept_name) VALUES
  (10, 'IT'),
  (20, 'Finance'),
  (30, 'HR'),
  (40, 'Sales'),
  (50, 'R&D'); -- intentionally has no employees to practice “zero” groups

INSERT INTO lab09.employee (emp_id, first_name, dept_id) VALUES
  (1, 'Amina',  10),
  (2, 'Omar',   10),
  (3, 'Lina',   20),
  (4, 'Noah',   30),
  (5, 'Sara',   NULL), -- no department
  (6, 'Yousef', 40);

-- Salaries:
-- - employee 5 has no salary row
-- - employee 2 has a salary row with NULL to demonstrate COUNT(col) and AVG(col)
INSERT INTO lab09.salary (emp_id, monthly_salary) VALUES
  (1, 90000.00),
  (2, NULL),
  (3, 85000.00),
  (4, 65000.00),
  (6, 72000.00);

INSERT INTO lab09.sales (dept_id, sale_month, amount) VALUES
  (10, '2026-01-01', 120000.00),
  (10, '2026-02-01',  80000.00),
  (20, '2026-01-01',  40000.00),
  (40, '2026-01-01',  60000.00),
  (40, '2026-02-01',  90000.00),
  (30, '2026-02-01',      0.00);

COMMIT;
```

Optional quick check:

```sql
SELECT 'department' AS table_name, count(*) AS rows FROM lab09.department
UNION ALL SELECT 'employee', count(*) FROM lab09.employee
UNION ALL SELECT 'salary', count(*) FROM lab09.salary
UNION ALL SELECT 'sales', count(*) FROM lab09.sales;
```

---

# 🧪 **Lab Exercises – Aggregate Functions**

---

## 🔹 **Step 1 – COUNT basics**

### 1.1 Count all employees (rows)

```sql
SELECT COUNT(*) AS total_employees
FROM lab09.employee;
```

### 1.2 COUNT(column) vs COUNT(*)

Count how many employees have a department vs total employees.

```sql
SELECT
  COUNT(*) AS total_employees,
  COUNT(dept_id) AS employees_with_department
FROM lab09.employee;
```

✅ **Observation:** `COUNT(dept_id)` ignores NULL `dept_id`.

---

## 🔹 **Step 2 – COUNT per group (GROUP BY)**

Count employees per department (only departments with at least one employee).

```sql
SELECT d.dept_name, COUNT(*) AS employee_count
FROM lab09.employee e
JOIN lab09.department d ON d.dept_id = e.dept_id
GROUP BY d.dept_name
ORDER BY employee_count DESC, d.dept_name;
```

---

## 🔹 **Step 3 – Keeping “zero” groups (LEFT JOIN)**

List *all* departments, including those with **zero** employees.

```sql
SELECT
  d.dept_name,
  COUNT(e.emp_id) AS employee_count
FROM lab09.department d
LEFT JOIN lab09.employee e ON e.dept_id = d.dept_id
GROUP BY d.dept_name
ORDER BY d.dept_name;
```

✅ **Observation:** `COUNT(e.emp_id)` is 0 for departments with no employees.

---

## 🔹 **Step 4 – SUM() and AVG()**

### 4.1 Total sales amount

```sql
SELECT SUM(amount) AS total_sales
FROM lab09.sales;
```

### 4.2 Sales total per department

```sql
SELECT d.dept_name, SUM(s.amount) AS dept_sales_total
FROM lab09.sales s
JOIN lab09.department d ON d.dept_id = s.dept_id
GROUP BY d.dept_name
ORDER BY dept_sales_total DESC;
```

### 4.3 Average sales amount per department

```sql
SELECT d.dept_name, ROUND(AVG(s.amount), 2) AS dept_sales_avg
FROM lab09.sales s
JOIN lab09.department d ON d.dept_id = s.dept_id
GROUP BY d.dept_name
ORDER BY dept_sales_avg DESC;
```

---

## 🔹 **Step 5 – MIN() and MAX()**

Find min/max sales amount per department.

```sql
SELECT d.dept_name,
       MIN(s.amount) AS min_sale,
       MAX(s.amount) AS max_sale
FROM lab09.sales s
JOIN lab09.department d ON d.dept_id = s.dept_id
GROUP BY d.dept_name
ORDER BY d.dept_name;
```

---

## 🔹 **Step 6 – Aggregates with JOINs (salary per department)**

Compute salary totals and averages per department.

```sql
SELECT
  d.dept_name,
  COUNT(e.emp_id) AS employees_in_dept,
  COUNT(sa.monthly_salary) AS employees_with_salary_value,
  SUM(sa.monthly_salary) AS total_salary,
  ROUND(AVG(sa.monthly_salary), 2) AS avg_salary
FROM lab09.department d
LEFT JOIN lab09.employee e ON e.dept_id = d.dept_id
LEFT JOIN lab09.salary sa ON sa.emp_id = e.emp_id
GROUP BY d.dept_name
ORDER BY d.dept_name;
```

✅ **Observation:** `COUNT(sa.monthly_salary)` ignores NULL salary values, while `COUNT(e.emp_id)` counts employees (even those without salary rows).

---

## 🔹 **Step 7 – WHERE vs HAVING**

### 7.1 Filter rows before aggregation (WHERE)

Total sales in February only.

```sql
SELECT SUM(amount) AS feb_total_sales
FROM lab09.sales
WHERE sale_month = '2026-02-01';
```

### 7.2 Filter groups after aggregation (HAVING)

Departments where total sales exceed 100000.

```sql
SELECT d.dept_name, SUM(s.amount) AS dept_sales_total
FROM lab09.sales s
JOIN lab09.department d ON d.dept_id = s.dept_id
GROUP BY d.dept_name
HAVING SUM(s.amount) > 100000
ORDER BY dept_sales_total DESC;
```

---

## 🔹 **Step 8 – Multi-column GROUP BY**

Sales totals by department and month.

```sql
SELECT d.dept_name, s.sale_month, SUM(s.amount) AS month_total
FROM lab09.sales s
JOIN lab09.department d ON d.dept_id = s.dept_id
GROUP BY d.dept_name, s.sale_month
ORDER BY d.dept_name, s.sale_month;
```

---

## 🔹 **Step 9 – Common mistake: selecting non-grouped columns**

In PostgreSQL, every selected column must be either:

- aggregated (e.g., `SUM(amount)`), or
- included in `GROUP BY`

✅ Correct pattern:

```sql
SELECT d.dept_name, COUNT(*) AS employee_count
FROM lab09.employee e
JOIN lab09.department d ON d.dept_id = e.dept_id
GROUP BY d.dept_name;
```

---

## 🧾 **Summary**

- `**COUNT(*)` vs `COUNT(col)`**: rows vs non-NULL values
- `**GROUP BY**`: one output row per group; selected columns must be grouped or aggregated
- `**WHERE**` filters rows **before** aggregation
- `**HAVING`** filters groups **after** aggregation
- Use **LEFT JOIN** to keep “zero” groups and `COALESCE(...)` when you need to display zeros instead of NULLs

---

## ✅ **Deliverables**

Submit **one `.sql` file** containing:

- Step 1.2 (COUNT(*) vs COUNT(column))
- Step 3 (departments with zero employees)
- Step 6 (salary aggregates per department)
- Step 7.2 (HAVING example)
- Step 8 (multi-column grouping)

---

## ⚙️ **Practice Exercises**

Use the `lab09` schema.

1. **Departments with the highest total sales**
  Return the department(s) with the highest `SUM(amount)`.
2. **Employees without a salary row**
  List employees who do not have a row in `lab09.salary`.
3. **Employees with salary above organization average**
  Count how many employees have a `monthly_salary` greater than the overall average salary (ignore NULL salaries).
4. **Departments with average salary below 80000**
  Include departments even if they have 0 salaries (treat “no salaries” as average = NULL and decide how you want to handle it).
5. **Month with highest total sales (stretch)**
  Return the `sale_month` with the highest total sales amount.

---

## 🧹 **Cleanup (avoid conflicts with the next lab)**

Run when you finish the lab:

```sql
DROP SCHEMA IF EXISTS lab09 CASCADE;
```


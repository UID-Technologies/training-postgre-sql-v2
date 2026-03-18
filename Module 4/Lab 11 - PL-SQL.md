
# 🧩 **Lab 11 – PL/pgSQL Programming (PostgreSQL Hands-on Lab)**

---

## 🎯 **Objectives**

By the end of this lab, learners will be able to:

* Explain the **PL/pgSQL language** and when to use it.
* Declare and use **variables and data types**.
* Implement **conditional logic** (`IF/ELSIF/ELSE`) and **loops** (`FOR`, `WHILE`).
* Create and execute **functions with IN/OUT parameters**.
* Implement **stored procedure patterns** for batch updates.
* Handle **exceptions** gracefully using `EXCEPTION WHEN`.
* Work with **cursors** for pagination and data iteration.

---

## 🧠 **What is PL/pgSQL?**

**PL/pgSQL (PostgreSQL Procedural Language)** is PostgreSQL’s built-in procedural language that allows logic to run inside the database engine.

You can:

- Declare **variables** and use strong **data types**.
- Add **control flow**: `IF`, `ELSIF`, `ELSE`, `LOOP`, `WHILE`, `FOR`.
- Build **functions** and **stored procedures** for shared business rules.
- Use **exception handling** and **cursors** for more advanced patterns.

It is ideal for operations that need to run close to the data (validations, batch jobs, common calculations).

---

## 📚 **PL/pgSQL Language Overview**

Key language elements you will practice in this lab:

- **Blocks**: anonymous (`DO $$ ... $$`) vs named (`CREATE FUNCTION` / `CREATE PROCEDURE`).
- **Variables and data types**: `DECLARE v_count int;`, `TEXT`, `NUMERIC`, etc.
- **Conditional statements**: `IF/ELSIF/ELSE`.
- **Loops**: `LOOP`, `WHILE`, and `FOR ... IN SELECT`.
- **Functions with parameters**: `IN`, `OUT`, and `INOUT`.
- **Exception handling**: `BEGIN ... EXCEPTION WHEN ... THEN ... END`.
- **Cursors**: `REFCURSOR`, `OPEN`, `FETCH`, `CLOSE` and cursor-style pagination.

---

## 🧰 **Setup**

1. Connect to your PostgreSQL container:

   ```bash
  docker exec -it postgres-container bash
   psql -U postgres
   ```

2. Create a fresh training database:

   ```sql
   CREATE DATABASE pllab;
   \c pllab;
   ```

3. Create a base schema and tables:

   ```sql
   CREATE SCHEMA hr;

   CREATE TABLE hr.employees (
     emp_id SERIAL PRIMARY KEY,
     full_name TEXT NOT NULL,
     dept TEXT,
     salary NUMERIC(10,2),
     rating INT DEFAULT 3
   );

   INSERT INTO hr.employees (full_name, dept, salary, rating) VALUES
     ('Asha Singh','IT',1200000,5),
     ('Rohit Verma','IT',950000,4),
     ('Karan Shah','Finance',650000,3),
     ('Pooja Das','Finance',450000,2),
     ('Neha Gupta','HR',500000,3);
   ```

---

# 🧪 **Lab Exercises – PL/pgSQL Concepts**

---

## 🔹 **Step 1 – Variables and Data Types**

### Goal

Learn to declare variables and assign values.

```sql
DO $$
DECLARE
  v_count INT;
  v_avg   NUMERIC(10,2);
BEGIN
  SELECT COUNT(*), AVG(salary) INTO v_count, v_avg FROM hr.employees;
  RAISE NOTICE 'Total employees = %, Average Salary = %', v_count, v_avg;
END$$;
```

✅ **Observation:**
Shows computed values using PL/pgSQL variables.

---

## 🔹 **Step 2 – Conditional Statements (IF/ELSIF/ELSE)**

```sql
DO $$
DECLARE
  v_rating INT := 4;
BEGIN
  IF v_rating >= 5 THEN
    RAISE NOTICE 'Excellent Performer';
  ELSIF v_rating >= 3 THEN
    RAISE NOTICE 'Meets Expectations';
  ELSE
    RAISE NOTICE 'Needs Improvement';
  END IF;
END$$;
```

✅ **Observation:**
Conditional logic executes based on variable value.

---

## 🔹 **Step 3 – Loops (FOR, WHILE)**

```sql
DO $$
DECLARE
  rec RECORD;
  counter INT := 1;
BEGIN
  -- WHILE loop
  WHILE counter <= 3 LOOP
    RAISE NOTICE 'WHILE Loop Counter = %', counter;
    counter := counter + 1;
  END LOOP;

  -- FOR loop over query
  FOR rec IN SELECT emp_id, full_name FROM hr.employees LOOP
    RAISE NOTICE 'Employee %: %', rec.emp_id, rec.full_name;
  END LOOP;
END$$;
```

✅ **Observation:**
Demonstrates iteration through counters and query results.

---

## 🔹 **Step 4 – Function with IN/OUT Parameters**

### Example – Bonus Calculation Function

```sql
CREATE OR REPLACE FUNCTION hr.calc_bonus(
  p_salary NUMERIC,
  p_rating INT,
  OUT bonus_amount NUMERIC
) LANGUAGE plpgsql AS $$
BEGIN
  IF p_rating >= 5 THEN
    bonus_amount := p_salary * 0.20;
  ELSIF p_rating = 4 THEN
    bonus_amount := p_salary * 0.12;
  ELSIF p_rating = 3 THEN
    bonus_amount := p_salary * 0.06;
  ELSE
    bonus_amount := 0;
  END IF;
END$$;

-- Test
SELECT full_name, salary, rating,
       hr.calc_bonus(salary, rating) AS bonus
FROM hr.employees;
```

✅ **Observation:**
Each employee’s bonus is computed based on rating.

---

## 🔹 **Step 5 – Exception Handling**

```sql
DO $$
DECLARE
  a INT := 10;
  b INT := 0;
  c NUMERIC;
BEGIN
  BEGIN
    c := a / b;
  EXCEPTION WHEN division_by_zero THEN
    RAISE NOTICE 'Divide-by-zero handled!';
    c := NULL;
  END;
  RAISE NOTICE 'Result = %', c;
END$$;
```

✅ **Observation:**
No crash — error handled gracefully.

---

## 🔹 **Step 6 – Working with Cursors**

```sql
DO $$
DECLARE
  cur REFCURSOR;
  rec RECORD;
BEGIN
  OPEN cur FOR SELECT emp_id, full_name, salary FROM hr.employees ORDER BY emp_id;
  LOOP
    FETCH cur INTO rec;
    EXIT WHEN NOT FOUND;
    RAISE NOTICE 'EmpID:% Name:% Salary:%', rec.emp_id, rec.full_name, rec.salary;
  END LOOP;
  CLOSE cur;
END$$;
```

✅ **Observation:**
Fetches rows one by one using cursor.

---

## 🔹 **Step 7 – Stored Procedure Pattern (Batch Payroll Update)**

```sql
CREATE OR REPLACE PROCEDURE hr.update_payroll(p_dept TEXT, p_raise NUMERIC)
LANGUAGE plpgsql AS $$
DECLARE
  v_count INT;
BEGIN
  UPDATE hr.employees
     SET salary = salary * (1 + p_raise)
   WHERE dept = p_dept;
  GET DIAGNOSTICS v_count = ROW_COUNT;
  RAISE NOTICE '% employees in % got a % %% raise', v_count, p_dept, p_raise * 100;
END$$;

-- Execute
CALL hr.update_payroll('IT', 0.05);
```

✅ **Observation:**
Procedure updates and reports affected rows.

---

## 🔹 **Step 8 – Cursor-Based Pagination**

```sql
CREATE OR REPLACE FUNCTION hr.paginate_employees(
  p_last_id INT DEFAULT 0,
  p_limit INT DEFAULT 2
) RETURNS TABLE(emp_id INT, full_name TEXT, salary NUMERIC) AS $$
BEGIN
  RETURN QUERY
  SELECT emp_id, full_name, salary
  FROM hr.employees
  WHERE emp_id > p_last_id
  ORDER BY emp_id
  LIMIT p_limit;
END$$ LANGUAGE plpgsql;

-- Test
SELECT * FROM hr.paginate_employees(0, 2);  -- page 1  
SELECT * FROM hr.paginate_employees(2, 2);  -- page 2
```

✅ **Observation:**
Implements key-set pagination using function.

---

## 🔹 **Step 9 – Handle Divide-By-Zero Gracefully**

```sql
CREATE OR REPLACE FUNCTION hr.safe_ratio(
  p_num NUMERIC,
  p_den NUMERIC
) RETURNS NUMERIC AS $$
DECLARE
  v_result NUMERIC;
BEGIN
  BEGIN
    v_result := p_num / p_den;
  EXCEPTION WHEN division_by_zero THEN
    v_result := NULL;
  END;
  RETURN v_result;
END$$ LANGUAGE plpgsql;

-- Test
SELECT hr.safe_ratio(10, 2); -- 5  
SELECT hr.safe_ratio(10, 0); -- NULL
```

✅ **Observation:**
Function handles arithmetic errors safely.

---

## 🔹 **Step 10 – Function with INOUT Parameter**

Use an `INOUT` parameter to both **accept** and **return** a value.

```sql
CREATE OR REPLACE FUNCTION hr.normalize_rating(
  INOUT p_rating INT
) LANGUAGE plpgsql AS $$
BEGIN
  IF p_rating IS NULL THEN
    p_rating := 3;           -- default rating
  ELSIF p_rating < 1 THEN
    p_rating := 1;
  ELSIF p_rating > 5 THEN
    p_rating := 5;
  END IF;
END$$;

-- Test
SELECT hr.normalize_rating(0)   AS r1,  -- becomes 1
       hr.normalize_rating(10)  AS r2,  -- becomes 5
       hr.normalize_rating(NULL) AS r3; -- becomes 3
```

✅ **Observation:**
The same parameter is used as both input and normalized output.

---

## 🔹 **Step 11 – Loop with CONTINUE and EXIT**

Demonstrate more advanced loop control using `CONTINUE` and `EXIT`.

```sql
DO $$
DECLARE
  rec hr.employees%ROWTYPE;
BEGIN
  FOR rec IN SELECT * FROM hr.employees ORDER BY emp_id LOOP
    -- Skip Finance department completely
    IF rec.dept = 'Finance' THEN
      CONTINUE;
    END IF;

    RAISE NOTICE 'Processing employee % (%), dept=%',
      rec.emp_id, rec.full_name, rec.dept;

    -- Stop once we have processed 3 non-Finance employees
    IF rec.emp_id >= 3 THEN
      EXIT;
    END IF;
  END LOOP;
END$$;
```

✅ **Observation:**
Shows how to skip specific rows and break out of a loop early.

---

## 🧾 **Summary**

- **PL/pgSQL Language**: blocks, variables, and strong typing.
- **Control flow**: `IF/ELSIF/ELSE` and loops (`WHILE`, `FOR`) for procedural logic.
- **Functions and procedures**: reusable units with `IN`/`OUT` parameters and `CALL`.
- **Exception handling**: `EXCEPTION WHEN` to catch and handle runtime errors.
- **Cursors and pagination**: row-by-row processing and key-set pagination patterns.

---

## ✅ **Deliverables**

Submit **one `.sql` file** containing:

- Step 1 (variables and data types example).
- Step 2 (IF/ELSIF/ELSE example).
- Step 4 (function with OUT parameter) and its test query.
- Step 7 (stored procedure pattern) and at least one `CALL`.
- Step 8 or 9 (a cursor or exception-handling function).

Screenshots are optional if required by your platform.

---

## 🧩 **Practice Challenges**

1. Write a function `hr.get_employee_bonus(dept TEXT)` that returns total bonus per department using the `hr.calc_bonus` logic.
2. Create a procedure that reduces salary by 10 % for employees with rating < 3, with proper exception handling for negative salaries.
3. Implement a cursor to print all departments and count employees in each (hint: `GROUP BY` + cursor over the grouped result).
4. Add a logging table and extend one function/procedure to **log errors or skipped rows** instead of only raising notices.

---

## 🧹 **Cleanup (optional, to avoid conflicts with future labs)**

When you are done with this lab, you can drop the training database:

```sql
DROP DATABASE IF EXISTS pllab;
```


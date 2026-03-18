# 🧩 **Lab 08 - SQL JOINs (PostgreSQL Hands-on Lab)**

---

## 🧭 **Introduction**

In real-world databases, data is normalized across multiple tables (employees, departments, salaries, projects). **JOINs** are how we recombine those related rows to answer business questions.

This lab focuses on:

- Building the *correct join predicate* (how tables relate)
- Choosing the right join type (INNER vs OUTER vs CROSS)
- Avoiding common mistakes (filters that unintentionally turn an OUTER JOIN into an INNER JOIN)

---

## 🎯 **Objectives**

By the end of this lab, learners will be able to:

- Explain **what a JOIN is** and when to use it
- Use **INNER**, **LEFT**, **RIGHT**, **FULL**, **SELF**, and **CROSS JOIN**
- Join **3+ tables** and handle **many-to-many** relationships
- Apply filters safely on joined queries (especially with OUTER JOINs)

---

## 🧠 **What is a JOIN?**

A **JOIN** combines rows from **two (or more) tables** into one result set, based on a relationship between columns (usually **foreign key ↔ primary key**).

Example relationship: employees belong to a department using `dept_id`.

---

## 🧱 **Common Types of JOINs**


| JOIN Type      | Best for                           | Returns                              |
| -------------- | ---------------------------------- | ------------------------------------ |
| **INNER JOIN** | Only matched relationships         | Only matching rows                   |
| **LEFT JOIN**  | Keep all rows from left            | All left rows + matched right rows   |
| **RIGHT JOIN** | Keep all rows from right           | All right rows + matched left rows   |
| **FULL JOIN**  | Keep everything                    | All rows from both sides             |
| **SELF JOIN**  | Compare rows within the same table | Pairs/relationships within one table |
| **CROSS JOIN** | Generate combinations              | Cartesian product (all pairs)        |


---

## 🧰 **Step 0 – Setup**

This lab creates its own schema (`lab08`) so it won’t conflict with previous/next labs.

Run the full script below once.

```sql
BEGIN;

DROP SCHEMA IF EXISTS lab08 CASCADE;
CREATE SCHEMA lab08;

CREATE TABLE lab08.department (
  dept_id   int PRIMARY KEY,
  dept_name text NOT NULL UNIQUE
);

CREATE TABLE lab08.employee (
  emp_id     int PRIMARY KEY,
  first_name text NOT NULL,
  email      text NOT NULL UNIQUE,
  dept_id    int NULL REFERENCES lab08.department(dept_id)
);

CREATE TABLE lab08.salary (
  emp_id         int PRIMARY KEY REFERENCES lab08.employee(emp_id),
  monthly_salary numeric(12,2) NOT NULL CHECK (monthly_salary > 0)
);

CREATE TABLE lab08.project (
  project_id   int PRIMARY KEY,
  project_name text NOT NULL UNIQUE
);

CREATE TABLE lab08.employee_project (
  emp_id     int NOT NULL REFERENCES lab08.employee(emp_id),
  project_id int NOT NULL REFERENCES lab08.project(project_id),
  role       text NOT NULL,
  PRIMARY KEY (emp_id, project_id)
);

INSERT INTO lab08.department (dept_id, dept_name) VALUES
  (10, 'IT'),
  (20, 'Finance'),
  (30, 'HR'),
  (40, 'Sales'),
  (50, 'R&D');

INSERT INTO lab08.employee (emp_id, first_name, email, dept_id) VALUES
  (1, 'Amina',   'amina@corp.test',   10),
  (2, 'Omar',    'omar@corp.test',    10),
  (3, 'Lina',    'lina@corp.test',    20),
  (4, 'Noah',    'noah@corp.test',    30),
  (5, 'Sara',    'sara@corp.test',    NULL), -- no department assigned
  (6, 'Yousef',  'yousef@corp.test',  40);

-- Give salaries to most, but not all employees (to practice OUTER JOINs)
INSERT INTO lab08.salary (emp_id, monthly_salary) VALUES
  (1, 90000.00),
  (2, 78000.00),
  (3, 85000.00),
  (4, 65000.00),
  (6, 72000.00);

INSERT INTO lab08.project (project_id, project_name) VALUES
  (100, 'Data Platform'),
  (200, 'ERP Upgrade'),
  (300, 'Payroll Automation');

-- Many-to-many assignments (note: Finance has a person with no project to practice later)
INSERT INTO lab08.employee_project (emp_id, project_id, role) VALUES
  (1, 100, 'Lead'),
  (2, 100, 'Developer'),
  (2, 200, 'Contributor'),
  (4, 300, 'Owner'),
  (6, 200, 'Analyst');

COMMIT;
```

Optional quick check:

```sql
SELECT 'department' AS table_name, count(*) AS rows FROM lab08.department
UNION ALL SELECT 'employee', count(*) FROM lab08.employee
UNION ALL SELECT 'salary', count(*) FROM lab08.salary
UNION ALL SELECT 'project', count(*) FROM lab08.project
UNION ALL SELECT 'employee_project', count(*) FROM lab08.employee_project;
```

---

# 🧪 **Lab Exercises – SQL JOINs**

---

## 🔹 **Step 1 – INNER JOIN**

Retrieve employees along with their department names (only rows where a department match exists).

```sql
SELECT e.emp_id, e.first_name, e.email, d.dept_name
FROM lab08.employee e
INNER JOIN lab08.department d
ON e.dept_id = d.dept_id;
```

✅ **Observation:**
Only employees with a valid department are shown.

---

### Example 2: Join 3 tables (employee + department + salary)

```sql
SELECT e.first_name, d.dept_name, s.monthly_salary
FROM lab08.employee e
INNER JOIN lab08.department d ON e.dept_id = d.dept_id
INNER JOIN lab08.salary s ON e.emp_id = s.emp_id;
```

✅ **Observation:**
Shows employees with both department and salary records.

---

## 🔹 **Step 2 – LEFT JOIN**

Return all employees even if they don’t have a department assigned.

```sql
SELECT e.first_name, e.dept_id, d.dept_name
FROM lab08.employee e
LEFT JOIN lab08.department d
ON e.dept_id = d.dept_id;
```

✅ **Observation:**
If a department is missing, `dept_name` will show `NULL`.

---

### Example 2: Employee and Salary (including employees without salary)

```sql
SELECT e.first_name, s.monthly_salary
FROM lab08.employee e
LEFT JOIN lab08.salary s
ON e.emp_id = s.emp_id;
```

✅ **Observation:**
Shows all employees; salary column will be `NULL` if not found.

---

## 🔹 **Step 3 – RIGHT JOIN**

Return all departments, even if no employees exist in them.

```sql
SELECT e.first_name, d.dept_name
FROM lab08.employee e
RIGHT JOIN lab08.department d
ON e.dept_id = d.dept_id;
```

✅ **Observation:**
All departments appear; employees may be `NULL` for empty departments.

💡 Trainer note: In PostgreSQL, `RIGHT JOIN` is valid, but many teams prefer writing the equivalent as a `LEFT JOIN` by swapping table order (often easier to read).

### Equivalent using LEFT JOIN (preferred style in many teams)

```sql
SELECT e.first_name, d.dept_name
FROM lab08.department d
LEFT JOIN lab08.employee e
  ON e.dept_id = d.dept_id;
```

---

## 🔹 **Step 4 – FULL JOIN**

Combine results of `LEFT` and `RIGHT` joins.
Shows all employees and departments — matched or not.

```sql
SELECT e.first_name, d.dept_name
FROM lab08.employee e
FULL JOIN lab08.department d
ON e.dept_id = d.dept_id;
```

✅ **Observation:**
All rows from both tables; unmatched ones have `NULL` values.

---

## 🔹 **Step 5 – SELF JOIN**

Use case: find employees in the same department (comparing a table with itself).

```sql
SELECT e1.first_name AS employee_name,
       e2.first_name AS colleague_name,
       e1.dept_id
FROM lab08.employee e1
JOIN lab08.employee e2
ON e1.dept_id = e2.dept_id
WHERE e1.emp_id <> e2.emp_id
ORDER BY e1.dept_id;
```

✅ **Observation:**
Pairs of employees who work in the same department.

---

## 🔹 **Step 6 – CROSS JOIN**

Returns all combinations (Cartesian product).
⚠️ Use carefully (can return a large number of rows).

```sql
SELECT e.first_name, p.project_name
FROM lab08.employee e
CROSS JOIN lab08.project p
ORDER BY e.first_name, p.project_name
LIMIT 15;
```

✅ **Observation:**
Each employee appears once for every project.

---

## 🔹 **Step 7 – Many-to-Many Join**

Retrieve employees and their assigned projects.

```sql
SELECT e.first_name, p.project_name, ep.role
FROM lab08.employee_project ep
JOIN lab08.employee e ON ep.emp_id = e.emp_id
JOIN lab08.project p ON ep.project_id = p.project_id
ORDER BY e.first_name, p.project_name;
```

✅ **Observation:**
Displays which employee works on which project and their role.

---

## 🔹 **Step 8 – Filtering Joined Results**

Get employees from **‘IT’ department** earning more than 80,000.

```sql
SELECT e.first_name, d.dept_name, s.monthly_salary
FROM lab08.employee e
JOIN lab08.department d ON e.dept_id = d.dept_id
JOIN lab08.salary s ON e.emp_id = s.emp_id
WHERE d.dept_name = 'IT' AND s.monthly_salary > 80000;
```

✅ **Observation:**
Shows filtered data after join conditions.

---

## 🔹 **Step 9 – OUTER JOIN pitfall: WHERE vs ON**

Goal: list **all departments**, and show employees if they exist, but only keep employees whose name starts with 'A'.

### Incorrect (turns into INNER JOIN)

```sql
SELECT d.dept_name, e.first_name
FROM lab08.department d
LEFT JOIN lab08.employee e
  ON e.dept_id = d.dept_id
WHERE e.first_name LIKE 'A%';
```

✅ **What happens:** the `WHERE` filter removes rows where `e.first_name` is `NULL`, which eliminates departments with no matching employees — effectively behaving like an INNER JOIN.

### Correct (filter in the join condition)

```sql
SELECT d.dept_name, e.first_name
FROM lab08.department d
LEFT JOIN lab08.employee e
  ON e.dept_id = d.dept_id
 AND e.first_name LIKE 'A%';
```

✅ **Observation:** all departments remain, and only qualifying employees appear.

---

## 🔹 **Step 10 – Cleaner joins with USING**

If both tables share the same join key name, `USING` can be more readable than `ON`, and it returns the join key **once** (instead of `table1.dept_id` and `table2.dept_id`).

```sql
SELECT e.emp_id, e.first_name, d.dept_name
FROM lab08.employee e
JOIN lab08.department d USING (dept_id);
```

✅ **Observation:** same result as an `INNER JOIN ... ON e.dept_id = d.dept_id`, but with simpler syntax when the column names match.

---

## 🧾 **Summary**

- **JOIN predicate**: connect tables using the relationship (`... ON e.dept_id = d.dept_id`)
- **INNER JOIN**: only matched rows
- **OUTER JOINs (LEFT/RIGHT/FULL)**: keep unmatched rows (watch out for `WHERE` filters!)
- **Many-to-many**: requires a bridge table (e.g., `employee_project`)
- **CROSS JOIN**: combinations; always control size with filters/limits

---

## ✅ **Deliverables**

Each learner should submit **one `.sql` file** containing:

- Step 1 (INNER JOIN)
- Step 2 (LEFT JOIN)
- Step 4 (FULL JOIN)
- Step 7 (Many-to-many join)
- Step 8 (Filtering after joins)
- Step 9 (Pitfall + corrected query)

Optional: screenshots are fine if required by your platform, but `.sql` is preferred for review and re-run.

---

## 🧩 **Practice Exercises**

Use the `lab08` schema.

1. **Employees without a department**
  List employees who do **not** have a department assigned.
2. **Departments with no employees**
  List departments that currently have **zero** employees.
3. **Total salary per department**
  Show `dept_name` and total salary. Include departments even if total salary is 0.  
   Hint: OUTER JOIN + `COALESCE`.
4. **Employees with no salary record**
  List employees missing a salary entry.
5. **Project staffing counts**
  For each project, show the number of assigned employees. Include projects with 0 employees.
6. **Finance employees with no project**
  Find employees working in **Finance** but not assigned to any project.
7. **Top paid per department (stretch)**
  Return the top-paid employee per department. If multiple tie, return all.

---

## 🧹 **Cleanup**

Run when you finish the lab:

```sql
DROP SCHEMA IF EXISTS lab08 CASCADE;
```


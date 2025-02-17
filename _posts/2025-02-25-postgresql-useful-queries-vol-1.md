---
title: 'PostgreSQL: Useful queries vol. 1'
tags: [PostgreSQL]
---

## PostgreSQL: Useful queries vol. 1

 - [List non-system tables](#list-non-system-tables)
 - [Foreign keys referencing the given column](#foreign-keys-referencing-the-given-column)
 - [Direct swap of data between two rows](#direct-swap-of-data-between-two-rows)
 - [Side-by-side comparison of monthly statistics of different years](#side-by-side-comparison-of-monthly-statistics-of-different-years)

---

---

### List non-system tables

```postgresql
SELECT table_schema || '.' || table_name AS tablename 
FROM information_schema.tables
WHERE 
    table_type = 'BASE TABLE'
    AND table_schema NOT IN ('pg_catalog', 'information_schema');
```

---

### Foreign keys referencing the given column

Imagine, you need to find out all the tables, that reference (as a foreign key) primary key on one particular table.
For example, you need to learn more about the design of new project database.

One might find useful this query for PostgreSQL which should return all references to the `id` attribute (column) 
of the `sports` relation (table):

```postgresql
SELECT
    conrelid::regclass AS table_name,
    conname AS foreign_key,
    pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE
    contype = 'f'
    AND connamespace = 'public'::regnamespace
    AND pg_get_constraintdef(oid) LIKE '% sports(id)'
ORDER BY 1, 2 DESC;
```

Output sample:

| table\_name | foreign\_key                 | definition                                        |
|-------------|------------------------------|---------------------------------------------------|
| tournaments | tournaments\_sport\_id\_fkey | FOREIGN KEY \(sport\_id\) REFERENCES sports\(id\) |

---

### Direct swap of data between two rows

Let us have a table with labels, but suddenly we have found a swapped colors between two of mostly commonly used
labels. Simple approach is to retrieve color code of both labels and then write two update queries. However,
one might choose error-prone way inspired by the following update query:

```postgresql
UPDATE labels AS l1
SET color_code = l2.color_code
FROM labels AS l2
WHERE
  l1.title IN ('good', 'excelent')
  AND
  l2.title IN ('good', 'excelent')
  AND
  l1.id <> l2.id;
```

---

### Side-by-side comparison of monthly statistics of different years

Let us have the following table with sample data:

```postgresql
CREATE TABLE daily_sales (
  date DATE,
  amount INTEGER
);

INSERT INTO daily_sales VALUES
  ('2023-02-05', 3000),
  ('2023-03-10', 2000),
  ('2023-03-21', 4000),
  ('2024-02-01', 2500),
  ('2024-02-20', 1500),
  ('2024-03-08', 3500);
```

Simpler way for retrieval of the data is here:

```postgresql
SELECT '2023-' || y23.month AS month, y23.total, '2024-' || y24.month AS month, y24.total
FROM 
  (
    SELECT 
      extract(MONTH FROM date) AS month,
      SUM(amount) as total
    FROM daily_sales
    WHERE extract(YEAR FROM date) = 2023
    GROUP BY extract(MONTH FROM date)
  ) AS y23
  INNER JOIN
  (
    SELECT
      extract(MONTH FROM date) as month,
      SUM(amount) as total
    FROM daily_sales
    WHERE extract(YEAR FROM date) = 2024
    GROUP BY extract(MONTH FROM date)
  ) AS y24
  ON y23.month = y24.month;
```

| month   | total | month   | total |
|---------|-------|---------|-------|
| 2023-02 | 3000  | 2024-02 | 4000  |
| 2023-03 | 6000  | 2024-03 | 3500  |

But one might always ask database engine to prepare a bit more, so application logic could be simpler.
The following query is an example of a bit more complex approach:

```postgresql
SELECT '2023-' || month.text AS month, y23.total, '2024-' || month.text AS month, y24.total
FROM 
  (
    SELECT
      generate_series(1, 12) AS number, 
      lpad(generate_series(1, 12)::text, 2, '0') AS text   
  ) AS month
  LEFT OUTER JOIN
  (
    SELECT
      extract(MONTH FROM date) AS month,
      SUM(amount) as total
    FROM daily_sales
    WHERE extract(YEAR FROM date) = 2023
    GROUP BY extract(MONTH FROM date)
  ) AS y23
  ON month.number = y23.month
  LEFT OUTER JOIN
  (
    SELECT
      extract(MONTH FROM date) as month,
      SUM(amount) as total
    FROM daily_sales
    WHERE extract(YEAR FROM date) = 2024
    GROUP BY extract(MONTH FROM date)
  ) AS y24
  ON month.number = y24.month
UNION
SELECT
  '2023-TOTAL',
  (
    SELECT SUM(amount) 
    FROM daily_sales
    WHERE extract(YEAR FROM date) = 2023
  ),
  '2024-TOTAL',
  (
    SELECT SUM(amount) 
    FROM daily_sales
    WHERE extract(YEAR FROM date) = 2024
  )
ORDER BY 1;
```

| month      | total | month      | total |
|------------|-------|------------|-------|
| 2023-01    |       | 2024-01    |       |
| 2023-02    | 3000  | 2024-02    | 4000  |
| 2023-03    | 6000  | 2024-03    | 3500  |
| 2023-04    |       | 2024-04    |       |
| 2023-05    |       | 2024-05    |       |
| 2023-06    |       | 2024-06    |       |
| 2023-07    |       | 2024-07    |       |
| 2023-08    |       | 2024-08    |       |
| 2023-09    |       | 2024-09    |       |
| 2023-10    |       | 2024-10    |       |
| 2023-11    |       | 2024-11    |       |
| 2023-12    |       | 2024-12    |       |
| 2023-TOTAL | 9000  | 2024-TOTAL | 7500  |

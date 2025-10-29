# Database queries

## Guidance and Best Practices

### Best practices and gotchas

When writing queries for use in FactoryTalk Optix or similar embedded database contexts, keep these recommendations in mind:

- Prefer explicit column lists instead of `SELECT *` in production code to avoid unexpected schema changes and reduce bandwidth.
- When using `DISTINCT`, be explicit about the columns you need — `DISTINCT *` removes duplicate entire rows which can be expensive.
- Avoid updating temporary tables when portability is a concern some backends restrict updates on temporary objects.
- Use table aliases (for example `t1`, `t2`) when joining multiple tables to prevent ambiguity and improve readability.
- Be cautious with deep subqueries or multiple nested levels - they can impact performance and readability.

### SQL compatibility and versions

Some features documented here are available starting from specific versions of FactoryTalk Optix (noted inline). Always test queries on the target database before deploying to production.

### Security & parameterization

FactoryTalk Optix always provides parameterized queries when inserting data to the database using the dedicated `Insert` method of the store. This helps prevent SQL injection attacks and ensures data integrity.

## SELECT queries

### Select All Columns

Retrieves all columns from a table.

```sql
SELECT * FROM TestTable1
```

### DISTINCT Qualifier

> [!NOTE]
> This feature is available starting from FactoryTalk Optix version 1.7.x.

The DISTINCT keyword eliminates duplicate rows from the result set. The ALL qualifier includes all rows, including duplicates (which is the default behavior).

```sql
SELECT DISTINCT Column1 FROM Table1
SELECT DISTINCT Column1, Column2 FROM Table1
SELECT DISTINCT * FROM Table1
```

### Select Specific Columns

Retrieves specific columns from a table.

```sql
SELECT ID, Name FROM TestTable1
```

### Column Aliases

Uses aliases to rename columns in the result set.

```sql
SELECT Name AS EmployeeName, Salary AS EmployeeSalary FROM TestTable1
```

### Filter Rows with Conditions

Filters rows based on conditions using WHERE clause.

```sql
SELECT * FROM TestTable1 WHERE Salary > 60000
SELECT * FROM TestTable1 WHERE Salary > 60000 AND DepartmentID = 101
SELECT * FROM TestTable1 WHERE DepartmentID = 101 OR DepartmentID = 102
SELECT * FROM TestTable1 WHERE NOT DepartmentID = 103
SELECT * FROM TestTable2 WHERE Location = 'Chicago'
SELECT Username FROM Users WHERE PrivateKey = 'test1234' AND Username <> 'test-user'
SELECT Username FROM Users WHERE PrivateKey = 'test1234' AND NOT (A = 6)
```

### IN

Filters rows where a column value matches any value in a specified list.

```sql
SELECT * FROM Table1 WHERE Column1 IN (10, 20, 30)
```

### BETWEEN

Filters rows where a column value falls within a specified range, inclusive of the boundaries.

```sql
SELECT * FROM Table1 WHERE Column1 BETWEEN 100 AND 200
```

### LIKE

Filters rows using pattern matching with wildcards. Supports escape characters for literal matching.

```sql
SELECT * FROM Table1 WHERE column1 LIKE '%a'
SELECT * FROM Table1 WHERE column1 LIKE '%a%'
SELECT * FROM Table1 WHERE column1 LIKE '%bbpi!%ppo%' ESCAPE '!'
```

### Sorting Results

Orders the result set by specified columns.

```sql
SELECT * FROM TestTable1 ORDER BY Salary ASC
SELECT * FROM TestTable1 ORDER BY Salary DESC
```

### Fully Qualified Identifiers in ORDER BY

> [!NOTE]
> This feature is available starting from FactoryTalk Optix version 1.7.x.

Supports fully qualified table.column references in the ORDER BY clause.

```sql
SELECT * FROM MyTable ORDER BY MyTable.MyColumn

SELECT Recipes.Name, RecipeMetadata_MyRecipeSchema.MyMetadata1 
FROM Recipes JOIN RecipeMetadata_MyRecipeSchema ON Recipes.Id = RecipeMetadata_MyRecipeSchema.RecipeId 
WHERE RecipeMetadata_MyRecipeSchema.MyMetadata1 = 'BB' 
ORDER BY Recipes.Name
```

### Pagination

Limits the number of rows returned and supports offset.

```sql
SELECT * FROM TestTable1 LIMIT 5
SELECT * FROM TestTable1 LIMIT 5 OFFSET 5
```

### Complex Filtering with Multiple Conditions

Combines multiple conditions with AND and OR operators.

```sql
SELECT * FROM TestTable1
WHERE (Salary > 60000 AND DepartmentID = 101) OR
(Salary < 50000 AND DepartmentID = 102)
```

### Combining * with Standard Column

Adds additional columns to the result set alongside all columns.

```sql
SELECT *, Timestamp FROM TestTable4
```

### Asterisked Identifiers

> [!NOTE]
> This feature is available starting from FactoryTalk Optix version 1.7.x.

Supports table.* syntax to select all columns from a specific table in JOIN queries.

```sql
SELECT A.*, B.* FROM Table1 AS A JOIN Table2 AS B on A.Id = B.Table1Id WHERE ...
```

### Combining * with Literal Values

Adds literal values as new columns alongside all existing columns.

```sql
SELECT *, 'StaticValue' AS LiteralColumn FROM TestTable4
```

### Special Characters in Identifiers

Handles column names with special characters using double quotes.

```sql
SELECT "Water level", ID FROM TestTable1
```

### CASE WHEN Expression

> [!NOTE]
> This feature is available starting from FactoryTalk Optix version 1.7.x.

The CASE WHEN expression allows conditional logic in SQL statements, supporting both SELECT and UPDATE operations.

```sql
SELECT 
	CASE 
		WHEN Id = 1 THEN 'A' 
		WHEN Id = 2 THEN 'B' 
	END 
FROM Table1 
WHERE Id IN (1, 2, 3)
```

```sql
SELECT 
	CASE 
		WHEN Id = 1 THEN 'A' 
		WHEN Id = 2 THEN 'B' 
		ELSE 'X' 
	END 
FROM Table1
```

```sql
UPDATE Table1 SET Value = 
	CASE 
		WHEN Id = 1 THEN 'A' 
		WHEN Id = 2 THEN 'B'
		ELSE 'X'
	END
```

```sql
UPDATE Table1 SET Value = 
	CASE 
		WHEN Id = 1 THEN 'A' 
		WHEN Id = 2 THEN 'B' 
	END 
WHERE Id IN (1, 2, 3)
```

### IS NULL and IS NOT NULL

Filters rows based on NULL values.

```sql
SELECT * FROM TestTable1 WHERE DepartmentID IS NULL
SELECT * FROM TestTable1 WHERE DepartmentID IS NOT NULL
```

### NOT

The NOT operator negates conditions and applies to all other operators, such as IN, BETWEEN, EXISTS, IS NULL, etc.

```sql
SELECT * FROM Table1 WHERE column1 IS NOT NULL
SELECT * FROM Table1 WHERE column1 NOT IN (10, 20)
SELECT * FROM Table1 WHERE column1 NOT BETWEEN 100 AND 200
```

### Basic Aggregates

Performs basic aggregation operations on columns.

```sql
SELECT COUNT(*) AS TotalCount FROM TestTable1
SELECT AVG(Salary) AS AverageSalary FROM TestTable1
SELECT SUM(Salary) AS TotalSalary FROM TestTable1
SELECT MAX(Salary) AS HighestSalary FROM TestTable1
SELECT MIN(Salary) AS LowestSalary FROM TestTable1
```

### Group Aggregates with Multiple Conditions

Groups rows by DepartmentID, counts employees and sums salaries for those with salary > 50000.

**Example Output:**

| DepartmentID | EmployeeCount | TotalSalary |
|--------------|---------------|-------------|
| 101          | 3             | 210000      |
| 102          | 2             | 105000      |

```sql
SELECT DepartmentID, COUNT(*) AS EmployeeCount, SUM(Salary) AS TotalSalary 
FROM TestTable1 
WHERE Salary > 50000 
GROUP BY DepartmentID
SELECT DepartmentID, AVG(Salary) AS AvgSalary
FROM TestTable1
GROUP BY DepartmentID
```

### Filter Aggregates with HAVING

Groups by department and calculates average salary, then filters groups where average salary > 65000.

**Example Output:**

| DepartmentID | AvgSalary |
|--------------|-----------|
| 101          | 67000     |
| 103          | 79000     |

```sql
SELECT DepartmentID, AVG(Salary) AS AvgSalary
FROM TestTable1
GROUP BY DepartmentID
HAVING AVG(Salary) > 65000
```

### Combining * with Aggregate Functions

Adds a total row count column to all columns from TestTable4 using window function.

**Example Output:**

| ID  | Timestamp           | EventName      | Duration | TotalCount |
|-----|---------------------|----------------|----------|------------|
| 1   | 1/1/2025 12:00 PM   | Maintenance    | 2.5      | 13         |
| 2   | 1/1/2025 12:15 PM   | Adjustment     | 1.5      | 13         |

```sql
SELECT *, COUNT(*) OVER () AS TotalCount FROM TestTable4
```

### INNER JOIN

Joins TestTable1 and TestTable2 on DepartmentID, returning employees with their department names.

**Input Tables:**

TestTable1 (Employees):

| ID | Name  | DepartmentID |
|----|-------|--------------|
| 1  | Alice | 101          |
| 2  | Bob   | 102          |

TestTable2 (Departments):

| DepartmentID | DepartmentName |
|--------------|----------------|
| 101          | Engineering    |
| 102          | Sales          |

**Output:**

| Name  | DepartmentName |
|-------|----------------|
| Alice | Engineering    |
| Bob   | Sales          |

```sql
SELECT t1.Name, t2.DepartmentName 
FROM TestTable1 AS t1 
INNER JOIN TestTable2 AS t2 
ON t1.DepartmentID = t2.DepartmentID
```

### INNER JOIN Without Table Name Aliases

Joins tables without using aliases.

```sql
SELECT TestTable1.Name, TestTable2.DepartmentName 
FROM TestTable1 
INNER JOIN TestTable2 
ON TestTable1.DepartmentID = TestTable2.DepartmentID
```

### Aggregates with INNER JOIN

Joins employees and departments, then groups by department name to count employees and average salary.

**Example Output:**

| DepartmentName | EmployeeCount | AvgSalary |
|----------------|---------------|-----------|
| Engineering    | 3             | 67000     |
| HR             | 2             | 79000     |
| IT             | 2             | 64000     |

```sql
SELECT t2.DepartmentName, COUNT(t1.ID) AS EmployeeCount, AVG(t1.Salary) AS AvgSalary 
FROM TestTable1 AS t1 
INNER JOIN TestTable2 AS t2 
ON t1.DepartmentID = t2.DepartmentID 
GROUP BY t2.DepartmentName
```

### LEFT JOIN

Includes all rows from left table and matching from right.

```sql
SELECT t1.Name, t2.Location FROM TestTable1 AS t1 LEFT JOIN TestTable2 AS t2 ON t1.DepartmentID = t2.DepartmentID
```

### LEFT JOIN with Filtering

Performs LEFT JOIN between employees and departments, then filters out rows where department name is NULL.

**Example Output:**

| Name    | DepartmentName |
|---------|----------------|
| Alice   | Engineering    |
| Bob     | Sales          |

```sql
SELECT t1.Name, t2.DepartmentName 
FROM TestTable1 AS t1 
LEFT JOIN TestTable2 AS t2 
ON t1.DepartmentID = t2.DepartmentID
WHERE t2.DepartmentName IS NOT NULL
```

### Aggregates with LEFT JOIN

LEFT JOIN employees and departments, group by department name, count employees and average salary. Includes departments with no employees (NULL counts).

**Explanation:** A LEFT JOIN returns all rows from the left table (TestTable1), along with matching rows from the right table (TestTable2). When no match is found, NULL values are included for columns from the right table.

```sql
SELECT t2.DepartmentName, 
       COUNT(t1.ID) AS EmployeeCount, 
       AVG(t1.Salary) AS AvgSalary 
FROM TestTable1 AS t1 
LEFT JOIN TestTable2 AS t2 
    ON t1.DepartmentID = t2.DepartmentID 
GROUP BY t2.DepartmentName
```

### CROSS JOIN

Creates Cartesian product of employees and departments, pairing each employee with every department.

**Example Output (Partial):**

| Name  | DepartmentName |
|-------|----------------|
| Alice | Engineering    |
| Alice | Sales          |
| Bob   | Engineering    |
| Bob   | Sales          |

**Explanation:** A CROSS JOIN produces a Cartesian product, where each row from the first table is paired with every row from the second table. This can result in a large dataset.

```sql
SELECT t1.Name, t2.DepartmentName 
FROM TestTable1 AS t1 
CROSS JOIN TestTable2 AS t2
```

### Subquery in FROM Clause

Uses a subquery to calculate average salaries per department, then filters departments with avg salary > 65000.

**Example Output:**

| DepartmentID | AvgSalary |
|--------------|-----------|
| 101          | 67000     |
| 103          | 79000     |

**Explanation:** The subquery calculates the average salary grouped by department, and the outer query filters the results to include only departments where the average salary exceeds 65,000.

```sql
SELECT * 
FROM (
    SELECT DepartmentID, AVG(Salary) AS AvgSalary 
    FROM TestTable1 
    GROUP BY DepartmentID
) AS SubQuery 
WHERE AvgSalary > 65000
```

### Subquery with JOIN

Joins employees with a subquery that calculates average salary per department.

**Example Output:**

| Name  | AvgSalary |
|-------|-----------|
| Alice | 67000     |
| Bob   | 55500     |

**Explanation:** The subquery calculates the average salary for each department. The main query then joins this result with TestTable1 to associate employee names with their respective department's average salary.

```sql
SELECT t1.Name, SubQuery.AvgSalary 
FROM TestTable1 AS t1 
INNER JOIN (
    SELECT DepartmentID, AVG(Salary) AS AvgSalary 
    FROM TestTable1 
    GROUP BY DepartmentID
) AS SubQuery 
ON t1.DepartmentID = SubQuery.DepartmentID
```

### Subquery in WHERE Clause for IN Condition

Filters employees whose salary is in the list of salaries from department 101.

**Example Output:**

| Name     |
|----------|
| Alice    |
| Charlie  |
| Grace    |

**Description:** Subqueries in the WHERE clause with the IN operator are supported. This query correctly filtered names where Salary matches the subquery result.

```sql
SELECT Name FROM TestTable1 WHERE Salary IN (SELECT Salary FROM TestTable1 WHERE DepartmentID = 101)
```

### Subquery in WHERE Clause with Comparison

> [!NOTE]
> This feature is available starting from FactoryTalk Optix version 1.7.x.

Supports subqueries in WHERE clause using comparison operators.

```sql
SELECT * FROM Table1 WHERE Column > (SELECT Value FROM Table2)
```

### EXISTS Operator

Finds employees in departments located in 'New York' using EXISTS subquery.

**Example Output:**

| Name     |
|----------|
| Alice    |
| Charlie  |
| Grace    |

**Description:** Queries with EXISTS are supported, correctly identifying employees in departments located in "New York."

```sql
SELECT Name FROM TestTable1 WHERE EXISTS (SELECT 1 FROM TestTable2 WHERE TestTable2.DepartmentID = TestTable1.DepartmentID AND TestTable2.Location = 'New York')
```

### Subquery in FROM Clause (Aggregation)

Uses subquery for aggregation in FROM.

```sql
SELECT AVG(Salary) AS AvgSalary FROM (SELECT * FROM TestTable1 WHERE DepartmentID = 101) AS SubQuery
```

### Multiple Subqueries with IN

Combines multiple IN subqueries.

```sql
SELECT Name FROM TestTable1 WHERE DepartmentID IN (SELECT DepartmentID FROM TestTable2 WHERE Location = 'New York') AND Salary IN (SELECT Salary FROM TestTable1 WHERE DepartmentID = 101)
```

### Multiple Subqueries with EXISTS

Combines multiple EXISTS subqueries.

```sql
SELECT Name FROM TestTable1 WHERE EXISTS (SELECT 1 FROM TestTable2 WHERE TestTable1.DepartmentID = TestTable2.DepartmentID AND TestTable2.Location = 'Chicago') AND EXISTS (SELECT 1 FROM TestTable1 AS Sub WHERE Sub.Salary > 60000 AND Sub.DepartmentID = 101)
```

### Mixed IN and EXISTS with Correlation

Mixes IN and EXISTS with correlation.

```sql
SELECT Name FROM TestTable1 WHERE DepartmentID IN (SELECT DepartmentID FROM TestTable2 WHERE Location = 'Chicago') AND EXISTS (SELECT 1 FROM TestTable2 WHERE TestTable2.DepartmentID = TestTable1.DepartmentID AND TestTable2.Location = 'New York')
```

### Subquery with Nested IN Clauses

Nested IN conditions in subquery.

```sql
SELECT Name FROM TestTable1 WHERE DepartmentID IN (SELECT DepartmentID FROM TestTable2 WHERE Location IN (SELECT Location FROM TestTable2 WHERE DepartmentID = 101))
```

### Subquery with Nested EXISTS Clauses

Nested EXISTS conditions.

```sql
SELECT Name FROM TestTable1 WHERE EXISTS (SELECT 1 FROM TestTable2 WHERE TestTable1.DepartmentID = TestTable2.DepartmentID AND EXISTS (SELECT 1 FROM TestTable1 AS Sub WHERE Sub.Salary > 70000))
```

### Deeper Nested Subquery with IN

Deeper nesting with IN.

```sql
SELECT Name FROM TestTable1 WHERE DepartmentID IN (SELECT DepartmentID FROM TestTable2 WHERE Location IN (SELECT Location FROM TestTable2 WHERE Location IN (SELECT Location FROM TestTable2 WHERE DepartmentID = 101)))
```

### Deeper Nested Subquery with EXISTS

Deeper nesting with EXISTS.

```sql
SELECT Name FROM TestTable1 WHERE EXISTS (SELECT 1 FROM TestTable2 WHERE TestTable1.DepartmentID = TestTable2.DepartmentID AND EXISTS (SELECT 1 FROM TestTable1 WHERE EXISTS (SELECT 1 FROM TestTable2 WHERE TestTable2.DepartmentID = TestTable1.DepartmentID)))
```

### Subquery with Nested IN Clauses and Conditions

Complex nested conditions.

```sql
SELECT Name FROM TestTable1 WHERE DepartmentID IN (SELECT DepartmentID FROM TestTable2 WHERE Location IN (SELECT Location FROM TestTable2 WHERE EXISTS (SELECT 1 FROM TestTable1 WHERE Salary > 70000)))
```

### Subquery in JOIN Condition with Explicit Alias

Subquery in JOIN ON clause.

```sql
SELECT t1.Name, t2.DepartmentName FROM TestTable1 AS t1 INNER JOIN (SELECT * FROM TestTable2 WHERE Location = 'Chicago') AS t2 ON t1.DepartmentID = t2.DepartmentID
```

### Validate Joins with Nested Subqueries and Alias

Nested subqueries in JOIN.

```sql
SELECT t1.Name FROM TestTable1 AS t1 INNER JOIN (SELECT DepartmentID FROM TestTable2 WHERE EXISTS (SELECT 1 FROM TestTable1 WHERE TestTable1.DepartmentID = TestTable2.DepartmentID)) AS t2 ON t1.DepartmentID = t2.DepartmentID
```

### Combined Subqueries and Joins with Explicit Alias

Combines subqueries and joins.

```sql
SELECT t1.Name, t2.DepartmentName FROM TestTable1 AS t1 LEFT JOIN (SELECT DepartmentID, DepartmentName FROM TestTable2 WHERE EXISTS (SELECT 1 FROM TestTable1 WHERE Salary > 60000)) AS t2 ON t1.DepartmentID = t2.DepartmentID
```

### Subqueries in FROM Clause (Filtered)

Filtered subquery in FROM.

```sql
SELECT * FROM (SELECT * FROM TestTable4 WHERE Duration > 2) AS SubQuery
```

### ROW_NUMBER Window Function

Assigns unique sequential numbers to rows within each department partition, ordered by salary descending.

**Example Output:**

| Name    | DepartmentID | Rank |
|---------|--------------|------|
| Grace   | 101          | 1    |
| Charlie | 101          | 2    |
| Alice   | 101          | 3    |

**Explanation:** The ROW_NUMBER function assigns a unique rank to each row within a partition (grouped by DepartmentID), ordered by Salary in descending order.

```sql
SELECT Name, DepartmentID, 
       ROW_NUMBER() OVER (PARTITION BY DepartmentID ORDER BY Salary DESC) AS Rank 
FROM TestTable1
```

### RANK Window Function

Assigns ranks within department partitions, handling ties by giving same rank to equal salaries.

**Explanation:** The RANK function assigns a rank to each row within a partition, ordered by Salary in descending order. Tied rows receive the same rank, with subsequent ranks skipping as needed.

```sql
SELECT Name, DepartmentID, 
       RANK() OVER (PARTITION BY DepartmentID ORDER BY Salary DESC) AS Rank 
FROM TestTable1
```

### SUM with PARTITION BY

Calculates total salary for each department and adds it to every row in that department.

**Example Output:**

| Name    | DepartmentID | TotalDeptSalary |
|---------|--------------|-----------------|
| Alice   | 101          | 201000          |
| Charlie | 101          | 201000          |

**Explanation:** The SUM function calculates the total Salary for all employees within each DepartmentID. This value is applied to every row in the respective partition.

```sql
SELECT Name, DepartmentID, 
       SUM(Salary) OVER (PARTITION BY DepartmentID) AS TotalDeptSalary 
FROM TestTable1
```

### AVG with PARTITION BY

Calculates averages within partitions.

```sql
SELECT Name, DepartmentID, 
       AVG(Salary) OVER (PARTITION BY DepartmentID) AS AvgDeptSalary 
FROM TestTable1
```

### Global Ranking Using ROW_NUMBER

Ranks all employees globally by salary in descending order.

**Example Output:**

| Name  | Salary | DepartmentID | GlobalRank |
|-------|--------|--------------|------------|
| Diana | 80000  | 103          | 1          |
| Ivy   | 78000  | 103          | 2          |
| Grace | 71000  | 101          | 3          |

**Explanation:** The ROW_NUMBER function assigns a unique global rank to each employee, ordered by salary in descending order. No partitioning is applied, so the ranking is across all rows.

```sql
SELECT Name, Salary, DepartmentID, 
       ROW_NUMBER() OVER (ORDER BY Salary DESC) AS GlobalRank 
FROM TestTable1
```

### Ranking Within Department: Lowest to Highest Salary

Ranks within department by ascending salary.

```sql
SELECT Name, DepartmentID, 
       ROW_NUMBER() OVER (PARTITION BY DepartmentID ORDER BY Salary ASC) AS RankLowToHigh 
FROM TestTable1
```

### ROW_NUMBER with Multiple Columns in ORDER BY

Ranks with tie-breaking by multiple columns.

```sql
SELECT Name, DepartmentID, 
       ROW_NUMBER() OVER (PARTITION BY DepartmentID ORDER BY Salary DESC, Name ASC) AS Rank 
FROM TestTable1
```

### Combining * with a Window Function

Adds window function column to all columns.

```sql
SELECT *, ROW_NUMBER() OVER (ORDER BY Timestamp) AS RowNum FROM TestTable4
```

### Extract Year, Month, Day, Hour, Minute, and Second

Extracts all datetime components from Timestamp column.

**Example Output (Partial):**

| ID  | Year | Month | Day | Hour | Minute | Second |
|-----|------|-------|-----|------|--------|--------|
| 1   | 2025 | 1     | 1   | 12   | 0      | 0      |
| 12  | 2025 | 2     | 28  | 23   | 59     | 59     |
| 13  | 2024 | 2     | 29  | 12   | 0      | 0      |

**Explanation:** The query uses the EXTRACT operator multiple times within the same SELECT statement to retrieve and display year, month, day, hour, minute, and second for each record in TestTable4.

```sql
SELECT ID, 
       EXTRACT(YEAR FROM Timestamp) AS Year, 
       EXTRACT(MONTH FROM Timestamp) AS Month, 
       EXTRACT(DAY FROM Timestamp) AS Day, 
       EXTRACT(HOUR FROM Timestamp) AS Hour, 
       EXTRACT(MINUTE FROM Timestamp) AS Minute, 
       EXTRACT(SECOND FROM Timestamp) AS Second 
FROM TestTable4
```

### Average Duration for Each Hour

Groups events by hour and calculates average duration for each hour.

**Example Output:**

| Hour | AvgDuration |
|------|-------------|
| 00   | 2.8         |
| 08   | 3.75        |
| 12   | 2.5625      |
| 23   | 2.93        |

**Explanation:** The AVG function calculates the average Duration for each hour extracted from the Timestamp. Grouping by Hour ensures the calculation is performed for each unique hour.

```sql
SELECT EXTRACT(HOUR FROM Timestamp) AS Hour, 
       AVG(Duration) AS AvgDuration 
FROM TestTable4 
GROUP BY Hour 
ORDER BY Hour
```

### Maximum Duration for Each Day

Groups events by day and finds maximum duration for each day.

**Example Output:**

| Day | MaxDuration |
|-----|-------------|
| 01  | 2.5         |
| 02  | 4.0         |
| 31  | 3.0         |

**Explanation:** The MAX function identifies the maximum Duration for each day extracted from the Timestamp. Grouping by Day ensures the calculation is performed for each unique day.

```sql
SELECT EXTRACT(DAY FROM Timestamp) AS Day, 
       MAX(Duration) AS MaxDuration 
FROM TestTable4 
GROUP BY Day 
ORDER BY Day
```

### Event Count by Month

Groups events by month and counts total events per month.

**Example Output:**

| Month | EventCount |
|-------|------------|
| 01    | 11         |
| 02    | 2          |

**Explanation:** The COUNT(*) function calculates the total number of events for each month extracted from the Timestamp. Grouping by Month ensures events are counted for each unique month.

```sql
SELECT EXTRACT(MONTH FROM Timestamp) AS Month, 
       COUNT(*) AS EventCount 
FROM TestTable4 
GROUP BY Month 
ORDER BY Month
```

### Grouping and Summing by Hourly Periods

Groups by year, month, day, hour and sums duration.

```sql
SELECT
    EXTRACT(YEAR FROM Timestamp) AS Year,
    EXTRACT(MONTH FROM Timestamp) AS Month,
    EXTRACT(DAY FROM Timestamp) AS Day,
    EXTRACT(HOUR FROM Timestamp) AS Hour,
    SUM(Duration) AS TotalDuration
FROM TestTable4
GROUP BY Year, Month, Day, Hour
ORDER BY Year, Month, Day, Hour
```

### Grouping by Hour and Summing Duration

Groups by hour and sums duration.

```sql
SELECT EXTRACT(HOUR FROM Timestamp) AS Hour, 
       SUM(Duration) AS TotalDuration 
FROM TestTable4 
GROUP BY Hour
```

### Grouping by Full Hour Periods

Groups by full hour periods using subquery.

```sql
SELECT Year, Month, Day, Hour, 
       SUM(Duration) AS TotalDuration
FROM (
    SELECT 
        EXTRACT(YEAR FROM Timestamp) AS Year,
        EXTRACT(MONTH FROM Timestamp) AS Month,
        EXTRACT(DAY FROM Timestamp) AS Day,
        EXTRACT(HOUR FROM Timestamp) AS Hour,
        Duration
    FROM TestTable4
) AS SubQuery
GROUP BY Year, Month, Day, Hour
ORDER BY Year, Month, Day, Hour
```

### Verifying Grouping and Ordering by Extracted Datetime Data in a Subquery with Non-Selected Columns

Groups by extracted data with additional columns.

```sql
SELECT EventName, 
       SUM(Duration) AS TotalDuration
FROM (
    SELECT 
        EXTRACT(YEAR FROM Timestamp) AS Year,
        EXTRACT(MONTH FROM Timestamp) AS Month,
        EXTRACT(DAY FROM Timestamp) AS Day,
        EXTRACT(HOUR FROM Timestamp) AS Hour,
        Duration,
        EventName
    FROM TestTable4
) AS SubQuery
GROUP BY Year, Month, Day, Hour
ORDER BY Year, Month, Day, Hour
```

### Grouping with Subquery and Extracted Values

Groups using subquery with extracted values.

```sql
SELECT Hour, 
       SUM(Duration) AS TotalDuration
FROM (
    SELECT EXTRACT(HOUR FROM Timestamp) AS Hour, 
           Duration
    FROM TestTable4
) AS SubQuery
GROUP BY Hour
ORDER BY Hour
```

### Filtering by EXTRACT in WHERE Clause (Year)

Filters by year using string comparison.

```sql
SELECT EXTRACT(HOUR FROM Timestamp) AS Hour, SUM(Duration) AS TotalDuration
FROM TestTable4
WHERE EXTRACT(YEAR FROM Timestamp) = '2025'
GROUP BY Hour
ORDER BY Hour
```

### Filtering by EXTRACT in WHERE Clause (Month)

Filters by month using string comparison.

```sql
SELECT EXTRACT(DAY FROM Timestamp) AS Day, AVG(Duration) AS AvgDuration
FROM TestTable4
WHERE EXTRACT(MONTH FROM Timestamp) = '01'
GROUP BY Day
ORDER BY Day
```

## UPDATE Queries

### Basic UPDATE

Simple single-row update (set Salary for a single Id):

```sql
UPDATE UpdateTest SET Salary = 56000 WHERE Id = 2
```

Update rows by matching text column (rename a person):

```sql
UPDATE UpdateTest SET Name = 'Eve-Updated' WHERE Name = 'Eve'
```

Update multiple rows using IN:

```sql
UPDATE UpdateTest SET Salary = Salary + 1000 WHERE Id IN (1, 3)
```

### Grouped UPDATE queries

These are used as a CASE/branching workaround when CASE expressions inside UPDATE may not be accepted by the store (FactoryTalk Optix versions prior to 1.7.x):

```sql
-- set High for very large salaries
UPDATE UpdateTest SET Status = 'High' WHERE Salary >= 80000

-- set Medium for mid-range salaries
UPDATE UpdateTest SET Status = 'Medium' WHERE Salary >= 65000 AND Salary < 80000

-- set Low for lower salaries
UPDATE UpdateTest SET Status = 'Low' WHERE Salary < 65000
```

### Update with CASE WHEN:

> [!NOTE]
> This feature is available starting from FactoryTalk Optix version 1.7.x.

```sql
UPDATE UpdateTest
SET Status = CASE
       WHEN Salary >= 80000 THEN 'High'
       WHEN Salary >= 65000 THEN 'Medium'
       ELSE 'Low'
END
```

If that raises a syntax error on your target (FactoryTalk Optix versions prior to 1.7.x), use the grouped-update workaround demonstrated earlier.

### UPDATE with schema-qualified table names

> [!NOTE]
> This feature is available starting from FactoryTalk Optix version 1.7.x.

Updates values in a table, supporting schema-qualified table names.

```sql
UPDATE schema1.table1 SET column1 = 3, ...
```

### Notes on unsupported or dialect-specific UPDATE forms

The following UPDATE forms were tested against the embedded database. Results are summarized and example workarounds are provided.

#### Subquery in SET (UPDATE ... SET col = (SELECT ...))

Attempting to set a column using a scalar subquery (for example, mapping values from another table in a single UPDATE statement) is currently not supported. 

Example of unsupported query (will fail):

```sql
UPDATE UpdateTest
SET Status = (SELECT NewStatus FROM StatusMap WHERE StatusMap.OldStatus = UpdateTest.Status)
WHERE Status IN (SELECT OldStatus FROM StatusMap)
```

Workaround: perform one simple UPDATE per mapping row (this is compatible and fast for small mapping tables):

```sql
-- mapping table has rows (OldStatus, NewStatus) = ('Active','Enabled'), ('Inactive','Disabled')
UPDATE UpdateTest SET Status = 'Enabled' WHERE Status = 'Active'
UPDATE UpdateTest SET Status = 'Disabled' WHERE Status = 'Inactive'
```

#### UPDATE ... FROM / join-style UPDATE

Some SQL dialects support `UPDATE ... FROM <join>` to update rows using a join with another table. These queries are not supported and will produce a syntax error.

Example of unsupported query (will fail):

```sql
UPDATE UpdateTest SET Status = StatusMap.NewStatus FROM StatusMap WHERE UpdateTest.Status = StatusMap.OldStatus
```

Workaround: use per-mapping UPDATE statements as shown above, or run a small client-side loop that queries mapping rows and issues single UPDATE statements for each mapping.

#### Correlated EXISTS in UPDATE WHERE

Running `UPDATE ... WHERE EXISTS (SELECT 1 FROM Other WHERE Other.key = Target.key ...)` is currently not supported. If you rely on correlated `EXISTS` for updates, translate to an explicit set of `WHERE` conditions or use per-row updates retrieved by a separate `SELECT`.

FactoryTalk Optix reliably accepts straightforward `UPDATE ... SET ... WHERE <condition>` forms. More advanced forms that embed subqueries inside the `SET` expression, use correlated `EXISTS` in `WHERE`, or use `UPDATE ... FROM` may fail depending on FactoryTalk Optix version. When unsupported, the recommended approach is to break the update into multiple simple `UPDATE` statements or perform the logic in the client and apply targeted `UPDATE`s.

## DELETE Queries

### Basic DELETE

Deletes rows from a table based on a condition.

```sql
DELETE FROM DeleteTestDemo WHERE id = 1
```

### Delete multiple rows using IN

```sql
DELETE FROM DeleteTestDemo WHERE id IN (2,3,4)
```

### Delete using a subquery

```sql
DELETE FROM DeleteTestDemo WHERE id IN (SELECT id FROM OtherTable WHERE Flag = 1)
```

### Delete all rows (use with caution)

```sql
DELETE FROM DeleteTestDemo
```

### DELETE using a subquery (non-correlated example)

```sql
DELETE FROM DeleteTestDemo WHERE GroupID IN (SELECT GroupID FROM OtherTable WHERE Flag = 1)
```

### Pattern and range deletes

```sql
DELETE FROM DeleteTestExtra2 WHERE Val LIKE 'a%'
DELETE FROM DeleteTestExtra2 WHERE Salary BETWEEN 50000 AND 53000
```

### Notes on unsupported or dialect-specific DELETE forms

During testing the server did not accept several DELETE variants:

- Correlated EXISTS deletes in the form `DELETE ... WHERE EXISTS (SELECT ... WHERE o.col = target.col ...)` are not supported.
- Multi-table DELETE / DELETE with JOIN (MySQL/SQL Server style) are not supported.
- CTE-based DELETE (`WITH ... DELETE ...`) are not supported.
- `DELETE ... RETURNING` and explicit transaction control (BEGIN/ROLLBACK) are not supported.

If you depend on any of the unsupported forms, ask for help translating to an equivalent supported pattern (for example, using `IN` with a subquery instead of a correlated `EXISTS`).

## INSERT Queries

`INSERT` queries are only allowed using the `Insert` method of the store. See the [Database interactions](./database-interaction.md) for details.

## Temporary Tables

Temporary tables provide a way to store intermediate results during query execution. Key characteristics include:

- Temporary tables, identified by the `##` prefix, can be created and accessed using double-quoted identifiers.
- Supported operations include querying, joining, and aggregations however, update operations are not permitted.
- Temporary tables can be dropped successfully, facilitating proper resource management.

### Temporary Tables with Quoted Names

Creates temporary table with quoted name.

```sql
CREATE TEMPORARY TABLE "##TempTable" AS SELECT DepartmentID, AVG(Salary) AS AvgSalary FROM TestTable1 GROUP BY DepartmentID
```

### Accessing Temporary Tables

Queries data from the temporary table created with department average salaries.

**Example Output:**

| DepartmentID | AvgSalary |
|--------------|-----------|
| 101          | 67000     |
| 102          | 55500     |
| 103          | 79000     |
| 104          | 64000     |
| 105          | 62000     |

```sql
SELECT * FROM "##TempTable"
```

### Joining Temporary Tables

Joins employees table with temporary table containing department average salaries.

**Example Output:**

| Name    | AvgSalary |
|---------|-----------|
| Alice   | 67000     |
| Bob     | 55500     |
| Charlie | 67000     |
| Diana   | 79000     |
| Eve     | 64000     |

```sql
SELECT t1.Name, t2.AvgSalary FROM TestTable1 AS t1 INNER JOIN "##TempTable" AS t2 ON t1.DepartmentID = t2.DepartmentID
```

### Dropping Temporary Tables

Removes temporary table.

```sql
DROP TABLE "##TempTable"
```

### Filtered Temporary Tables

Creates temporary table with filtered data.

```sql
CREATE TEMPORARY TABLE "##FilteredTemp" AS SELECT ID, Name, Salary FROM TestTable1 WHERE Salary > 60000
```

### Accessing Filtered Temporary Tables

Queries the temporary table filtered to employees with salary > 60000.

**Example Output:**

| ID | Name    | Salary |
|----|---------|--------|
| 3  | Charlie | 70000  |
| 4  | Diana   | 80000  |
| 5  | Eve     | 65000  |
| 7  | Grace   | 71000  |

```sql
SELECT * FROM "##FilteredTemp"
```

### Aggregations on Temporary Tables

Counts high-salary employees from the filtered temporary table.

**Example Output:**

| HighSalaryCount |
|-----------------|
| 7               |

```sql
SELECT COUNT(*) AS HighSalaryCount FROM "##FilteredTemp"
```

## Queries on UI objects

### Populate a PieChart with the count of unique values in a column

This query returns the count of unique values in the column `Code` of the table `SQLiteStoreTable1` to populate a pie chard. Each slice of the pie chart will represent a unique value in the column `Code` and the size of the slice will be the count of the occurrences of that value.

The PieChard object should be populated with:

- Model: `EmbeddedDatabase1`
- Query: `SELECT Code, COUNT(*) AS Count FROM SQLiteStoreTable1 GROUP BY Code ORDER BY Count DESC`
- Label: `{Item}/Code`
- Value: `{Item}/Count`

### Populate a Histogram chart with the count of unique values in a column

This query returns the count of unique values in the column `StatusMachine` of the table `SQLiteStoreTable1` to populate a histogram chart. Each bar of the histogram will represent a unique value in the column `StatusMachine` and the height of the bar will be the count of the occurrences of that value.

- Model: `EmbeddedDatabase1`
- Query: `SELECT StatusMachine, COUNT(*) AS Occurrences FROM Machine_state WHERE StatusMachine >= 0 GROUP BY StatusMachine ORDER BY Occurrences DESC`
- Label: `{Item}/StatusMachine`
- Value: `{Item}/Occurrences`

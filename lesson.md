# **Lesson Plan: Advanced SQL for Business Insights**

### Lesson Overview

This lesson introduces advanced query and statements. Learners will be able to use meta queries to retrieve information about the database, use joins and unions to combine data from multiple tables, use window functions to calculate aggregates over a set of rows, use subqueries, and apply common table expressions to create temporary tables.

---

## **Part 1: The Map and the Bridge (Joins & Unions)** 

### **Learning Objectives**

* Navigate database schemas using Meta Queries.  
* Combine data using 4 Join types and Unions.

### **Theory Recap**

**The Analogy:** Imagine you are at a party. You have a list of names (client) and a list of who brought which gift (claim).

* **Inner Join:** Only people at the party who brought a gift.  
* **Left Join:** Everyone at the party; if they didn't bring a gift, the "gift" column is just an empty box (NULL).  
* **Union:** Putting two lists of names (e.g., Employees and Contractors) into one long "Staff" list.

### **Workshop**

Open DbGate and create a new connection to the DuckDB database file `db/unit-1-5.db`.

The tables we will be using are in the `main` (default) schema.

**Analogy:** "Before we build the bridge, we need to know the terrain. Let's look at our 'Card Catalog'."

#### List and describe tables
To list all the tables in `main`, run the following query:

```sql
SHOW TABLES;
```

You should see 4 rows of data. Each row represents the name of a table in the schema:

- `address`
- `car`
- `claim`
- `client`

If you want to see more details, run:

```sql
SHOW ALL TABLES;
```

To view the schema of an individual table, use the `DESCRIBE` command.

```sql
DESCRIBE address;
```

You should see the column names and data types.

**Exercie:** Describe the other 3 tables. Study their column names and data types.

#### Summarize tables

You can use the `SUMMARIZE` command to launch a query that computes a number of aggregates over all columns (of a table or query), including min, max, avg, std and approx_unique.

```sql
SUMMARIZE address;
```

**Exercise:** Summarize the other 3 tables. Study their min, max, approx_unique, avg and std (if applicable).

#### Joins and Unions

Joins are used to combine data from multiple tables. They are useful when you want to query data that is spread across multiple tables. You can join tables based on a common column, usually the _primary key_ of one table and the _foreign key_ of another table.

Let's look at the ERD of the database:

```dbml
Table claim {
  id int [pk]
  claim_date varchar
  travel_time int
  claim_amt int
  motor_vehicle_record int
  car_id int
  client_id int
}

Table car {
  id int [pk]
  resale_value int
  car_type varchar
  car_use varchar
  car_manufacture_year int
}

Table client {
  id int [pk]
  first_name varchar
  last_name varchar
  email varchar
  phone varchar
  birth_year int
  education varchar
  gender varchar
  home_value int
  home_kids int
  income int
  kids_drive int
  marriage_status varchar
  occupation varchar
  address_id int
}

Table address {
  id int [pk]
  country varchar
  state varchar
  city varchar
}


Ref: claim.car_id > car.id

Ref: claim.client_id > client.id

Ref: client.address_id > address.id
```

![ERD](./assets/erd.png)

`car_id`, `client_id` and `address_id` are foreign keys.

The 4 common types of joins are:

- Inner join
- Left join
- Right join
- Full (Outer) join

An overview of the different types of joins:

![Joins](./assets/join_types.png)

#### Inner join

An inner join returns only the rows that match in both tables.

```sql
SELECT *
FROM claim
INNER JOIN car ON claim.car_id = car.id;
```

You can also use the `JOIN` keyword instead of `INNER JOIN`. They are the same. But it's good practice to use `INNER JOIN` to make your query more readable.

> Inner join claim and client.
>
> Inner join client and address.

#### Left join

A left join returns all the rows from the left table, and the matching rows from the right table.

```sql
SELECT *
FROM claim
LEFT JOIN car ON claim.car_id = car.id;
```

#### Right join

A right join returns all the rows from the right table, and the matching rows from the left table.

```sql
SELECT *
FROM claim
RIGHT JOIN car ON claim.car_id = car.id;
```

#### Full (outer) join

A full join returns all the rows from both tables.

```sql
SELECT *
FROM claim
FULL JOIN car ON claim.car_id = car.id;
```
> Return a joined table containing `id, claim_date, travel_time, claim_amt` from claim, `car_type, car_use` from car, `first_name, last_name` from client and `state, city` from address.


#### Union

A union combines the results of two or more tables or queries into a single result set. The queries must have the same number of columns and compatible data types.

![Joins](./assets/join_vs_union.png)

It is not useful for this database, but assuming we have an employees table with the following columns:

- id
- name
- email
- phone

And a contractors table with the same columns. We can use a union to combine the two tables: 
**Note: the following code is to show the syntax only**

```sql
SELECT *
FROM employees
UNION
SELECT *
FROM contractors;
```

`UNION` removes duplicate rows. If you want to keep duplicate rows, use `UNION ALL` instead.

**Questions:** "If I SUMMARIZE the claim table and see a max(claim_amt) that is 10x higher than the average, what does that tell you about our insurance risk?"

**Exercise:**

Create a master report of every claim. Include the client's name, their car type, and the city they live in.  
Hint: You will need to join 4 tables.  

<details>

  <summary>Solution for Master Report</summary>
  
```sql
-- Solution for Master Report  
SELECT   
    cl.id, cl.claim_date, cl.claim_amt,  
    c.car_type,  
    cli.first_name, cli.last_name,  
    a.city, a.state  
FROM claim cl  
INNER JOIN car c ON cl.car_id = c.id  
INNER JOIN client cli ON cl.client_id = cli.id  
INNER JOIN address a ON cli.address_id = a.id;  
```
</details>


### **Q\&A / Reflection**

**Anticipated Hurdle:** Learners often struggle with "Which table is Left?".  

**Solution:** Always think of the "Primary Subject" as the Left table.  

**Business Case:** How would a "Full Outer Join" help us find cars that have never been claimed AND claims that (erroneously) don't have a car attached?


## **Part 2: The Moving Window (Window Functions)**

### **Learning Objectives**

* Calculate running totals and rankings without losing row detail.

### **Theory Recap**

The Analogy:  
Standard GROUP BY is like taking a whole class and saying "The average height is 5'8"." You lose the individuals.  
A Window Function is like walking down the line of students and saying "You are the 1st tallest, you are the 2nd tallest..." while they all remain standing in line.

### **Workshop**

### Window functions

Window functions are used to compute aggregates over a group of rows, called a window. They are useful when you want to compute aggregates over a group of rows, but you don't want to group the rows into a single output row. For example, you can use window functions to compute the running total of a column. You can also use window functions to compute the rank of a row. Window functions are also called analytical functions.

#### Running total

A running total is the sum of values in a column, where the sum is accumulated over time. For example, the running total of the following column:

```
1
2
3
4
```

is:

```
1
3
6
10
```

You can use the `SUM` window function to compute the running total of a column. The `SUM` window function takes a column as input, and returns the sum of the column values in the current row and all previous rows.

```sql
SELECT
  id, claim_amt,
  SUM(claim_amt) OVER (ORDER BY id) AS running_total
FROM claim;
```

The `OVER` clause defines the window. The `ORDER BY` clause defines the order of the rows in the window. The `SUM` window function computes the running total of the `claim_amt` column.

`PARTITION BY` can be used to define the groups in the window. For example, if we want to compute the running total of the `claim_amt` column for each `car_id`:

```sql
SELECT
  id, car_id, claim_amt,
  SUM(claim_amt) OVER (PARTITION BY car_id ORDER BY id) AS running_total
FROM claim;
```

> Return a table containing `id, car_id, claim_amt, running_total` from claim, where `running_total` is the running sum of the `claim_amt` column for each `car_id`.

#### Rank

The `RANK` window function computes the rank of a row in an ordered window. The rank of a row is the number of rows that come before the row, plus one. For example, the rank of the following column:

```
a
b
c
d
```

is:

```
1
2
3
4
```

You can use the `RANK` window function to compute the rank of a row. The `RANK` window function takes a column as input, and returns the rank of the column value in the current row.

```sql
SELECT
  id, car_id, claim_amt,
  RANK() OVER (PARTITION BY car_id ORDER BY claim_amt DESC) AS rank
FROM claim;
```

The `OVER` clause defines the window. The `PARTITION BY` clause defines the groups in the window. The `ORDER BY` clause defines the order of the rows in the window. The `RANK` window function computes the rank of the `claim_amt` column. It gives the same rank to rows with the same column value (`claim_amt`).

To return a different rank for each row, use the `ROW_NUMBER` window function instead.

> Return a table containing `id, car_id, travel_time, rank` from claim, where `rank` is the rank of the `travel_time` in descending order for each `car_id`.


#### Qualify

The `QUALIFY` clause is used to filter rows in a window. It is useful when you want to filter rows based on the result of a window function. For example, if we want to return the rows with a rank of 1:

"Let's find our 'Heavy Hitters'. Who has the highest claims per car category?"

```sql

-- The Ranking Window  
SELECT   
    id, car_id, claim_amt,  
    RANK() OVER (PARTITION BY car_id ORDER BY claim_amt DESC) AS rank  
FROM claim  
QUALIFY rank = 1; -- DuckDB specific 'QUALIFY' to filter windows  
```

Hands-on Exercise 2:  
Calculate a running total of insurance payouts over time (ordered by claim_date).  

```sql
-- Solution for Running Total  
SELECT   
    claim_date, claim_amt,  
    SUM(claim_amt) OVER (ORDER BY claim_date) AS running_total  
FROM claim;  
```

## **Part 3: Nested Logic (Subqueries & CTEs)**

### **Learning Objectives**

* Simplify complex logic using Common Table Expressions (CTEs).

### **Theory Recap**

The Analogy:  
A Subquery is like a "thought within a thought."  
A CTE (Common Table Expression) is like writing down a recipe step before you start cooking. It makes the code readable for humans, not just machines.

### **Workshop**

#### Subqueries

A subquery is a query nested inside another query. It is useful when you want to use the result of a query as input to another query.

For example, if we want to find the cars that have been involved in a claim:

```sql
SELECT id, resale_value, car_type
FROM car
WHERE id IN (
  SELECT DISTINCT car_id
  FROM claim
);
```

#### Correlated subquery

A correlated subquery is a subquery that references a column from the outer query. It is useful when you want to use the result of a query as input to another query, and the inner query depends on the outer query. The subquery is evaluated once for each row processed by the outer query.

For example, if we want to find the cars that have been involved in a claim, and the claim amount is greater than 10% of the car's resale value:

```sql
SELECT id, resale_value, car_type
FROM car c
WHERE id IN (
  SELECT DISTINCT car_id
  FROM claim
  WHERE claim_amt > 0.1 * c.resale_value
);
```

You can use the `EXISTS` operator to check if a subquery returns any rows. It is useful when you want to check if a subquery returns any rows, and the result of the query doesn't matter.

For example, if we want to find the cars that have been involved in a claim:

```sql
SELECT id, resale_value, car_type
FROM car c1
WHERE EXISTS (
  SELECT DISTINCT car_id
  FROM claim c2
  WHERE c2.car_id = c1.id
);
```

The `EXISTS` operator returns true if the subquery returns any rows, and false otherwise.

#### Subquery in FROM

A subquery in the `FROM` clause is called a derived table. It is useful when you want to use the result of a query as a table.

For example, if we want to find the cars that have been involved in a claim, and the car resale value is less than the average resale value for the car type:

```sql
SELECT id, resale_value, c1.car_type
FROM car c1
INNER JOIN (
  SELECT car_type, AVG(resale_value) AS average_resale_value
  FROM car
  GROUP BY car_type
) c2 ON c1.car_type = c2.car_type
WHERE resale_value < average_resale_value;
```

The derived table is useful when you want to use the result of a query as a table.

> Return a table containing `id, resale_value, car_use` from car, where the car resale value is less than the average resale value for the car use.

#### Common Table Expressions

A common table expression (CTE) is a named subquery. It is useful when you want to use the result of a query as input to another query, and the subquery is used more than once. The CTE is evaluated once, and the result is stored in a temporary table. The temporary table can be referenced in the query.

Using the same example as above:

```sql
WITH avg_resale_value_by_car_type AS (
  SELECT car_type, AVG(resale_value) AS average_resale_value
  FROM car
  GROUP BY car_type
)
SELECT id, resale_value, c1.car_type
FROM car c1
INNER JOIN avg_resale_value_by_car_type c2 ON c1.car_type = c2.car_type
WHERE resale_value < average_resale_value;
```

Let's find cars whose resale value is below the average for their specific type. It sounds complex, but we'll build it layer by layer.

```sql
-- The CTE Approach (Clean and Readable)  
WITH AvgValues AS (  
    SELECT car_type, AVG(resale_value) as avg_resale  
    FROM car  
    GROUP BY car_type  
)  
SELECT c.id, c.car_type, c.resale_value, a.avg_resale  
FROM car c  
JOIN AvgValues a ON c.car_type = a.car_type  
WHERE c.resale_value < a.avg_resale;
```

**Exercise:**  
Find clients who have made claims that are more than 50% of their annual income. Use a CTE to calculate the total claims per client first.

### **Q\&A / Reflection**

**Reflection:** Why do developers prefer CTEs over Subqueries? (Answer: Readability and Debugging).

## **Optional Topics for Self Study**

* CROSS JOIN & SELF JOIN (Theoretical understanding).

* UNION vs UNION ALL (Briefly touched upon if time permits).

* Advanced SUMMARIZE statistics (std, approx_unique).


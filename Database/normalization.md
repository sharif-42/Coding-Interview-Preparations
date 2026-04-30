# Database Normalization

Database normalization is the process of organizing data into smaller, well-structured tables so that duplicate data is reduced and data stays consistent.

In simple terms, normalization means:

- store each fact in one proper place,
- avoid repeating the same data again and again,
- and connect related data using keys.

For example, instead of storing department information in every student row, we store department details in a separate `departments` table and connect students to departments using `department_id`.

Normalization is mainly used in relational databases such as MySQL, PostgreSQL, SQL Server, and Oracle.

---

## 1. Pros of Normalization

### Reduces data redundancy

The same information is not stored repeatedly in many rows.

Example: department name and HOD name do not need to be repeated for every student.

### Improves data consistency

When data is stored in one place, updating it becomes safer.

Example: if a department name changes, you update it once instead of updating hundreds of rows.

### Prevents anomalies

Normalization helps avoid three common problems:

- **Update anomaly:** same value must be changed in many places
- **Insert anomaly:** you cannot insert one fact without another unrelated fact
- **Delete anomaly:** deleting one row accidentally removes important information

### Makes maintenance easier

Smaller, cleaner tables are easier to understand and maintain.

### Saves storage

Repeated values take unnecessary space. Removing duplication can reduce storage use.

---

## 2. Cons of Normalization

### More tables

After normalization, one large table is often split into multiple smaller tables.

### More joins in queries

To fetch complete information, queries often need `JOIN` operations.

Example: to get student name, department name, and course name, you may need to join multiple tables.

### Queries can become more complex

Highly normalized schemas can be harder for beginners to query and understand.

### Read performance can be slower in some cases

For reporting or analytics, too many joins may slow down reads compared to a denormalized structure.

That is why some systems normalize for correctness first, then selectively denormalize for performance later.

---

## 3. Why We Need Normalization

We need normalization because badly designed tables create duplicate data and inconsistent data.

Consider this table:

| student_id | student_name | department_name | hod_name |
|---|---|---|---|
| 1 | Alice | CSE | Dr. Khan |
| 2 | Bob | CSE | Dr. Khan |
| 3 | Carol | EEE | Dr. Sen |

Problems:

- `CSE` and `Dr. Khan` are repeated in multiple rows.
- If `Dr. Khan` changes to `Dr. Rahman`, every `CSE` row must be updated.
- If one row is missed, the database becomes inconsistent.
- If all CSE students are deleted, department information may also be lost.

Normalization solves this by separating student data from department data.

---

## 4. When We Need Normalization

Normalization is needed when:

- the same data is repeated many times,
- updates are becoming error-prone,
- the database is used for transactional systems,
- data consistency is more important than raw read speed,
- and the schema is growing and becoming difficult to manage.

Typical use cases:

- banking systems
- e-commerce order systems
- school or university management systems
- HR and payroll systems
- inventory management systems

These systems require correct and consistent data, so normalization is important.

### When normalization may not be enough by itself

In analytics or reporting systems, sometimes denormalization is used for faster reads. But the base design is usually still created using normalization principles first.

---

## 5. Definitions of 1NF, 2NF, and 3NF

### First Normal Form (1NF)

A table is in 1NF if:

- each column contains atomic values,
- there are no repeating groups or multi-valued columns,
- and each row can be identified uniquely.

In short, 1NF means one value per cell and no repeated sets of columns.

### Second Normal Form (2NF)

A table is in 2NF if:

- it is already in 1NF,
- and every non-key attribute depends on the whole primary key, not just part of it.

This mainly matters when the primary key is composite. 2NF removes partial dependency.

### Third Normal Form (3NF)

A table is in 3NF if:

- it is already in 2NF,
- and non-key attributes do not depend on other non-key attributes.

This removes transitive dependency, meaning non-key data should depend only on the key.

---

## 6. Explanation with Example

Let us start with an unnormalized table.

### Unnormalized Table

| order_id | customer_name | customer_city | product_1 | product_2 |
|---|---|---|---|---|
| 101 | Alice | Dhaka | Keyboard | Mouse |
| 102 | Bob | Sylhet | Monitor | NULL |

This table has several problems:

- multiple products are stored in separate columns
- customer information is repeated
- the design does not scale well if an order has many products

Now let us normalize it step by step.

---

### First Normal Form (1NF)

In 1NF, every column should contain atomic values, which means one value per cell.

So we rewrite the table like this:

| order_id | customer_name | customer_city | product |
|---|---|---|---|
| 101 | Alice | Dhaka | Keyboard |
| 101 | Alice | Dhaka | Mouse |
| 102 | Bob | Sylhet | Monitor |

This is better because there are no repeating product columns such as `product_1`, `product_2`, and so on.

But there is still redundancy:

- `Alice` and `Dhaka` are repeated for the same order

---

### Second Normal Form (2NF)

Suppose the key is `(order_id, product)`.

Now notice:

- `customer_name` depends only on `order_id`
- `customer_city` depends only on `order_id`

They do not depend on the full key `(order_id, product)`, so this is a partial dependency.

To fix that, split the table into:

**Orders**

| order_id | customer_name | customer_city |
|---|---|---|
| 101 | Alice | Dhaka |
| 102 | Bob | Sylhet |

**Order_Items**

| order_id | product |
|---|---|
| 101 | Keyboard |
| 101 | Mouse |
| 102 | Monitor |

Now product information is separated from order-level information.

---

### Third Normal Form (3NF)

Now look at the `Orders` table:

| order_id | customer_name | customer_city |
|---|---|---|
| 101 | Alice | Dhaka |
| 102 | Bob | Sylhet |

If one customer places many orders, `customer_name` and `customer_city` will be repeated again.

To improve this, create a separate `Customers` table.

**Customers**

| customer_id | customer_name | customer_city |
|---|---|---|
| 1 | Alice | Dhaka |
| 2 | Bob | Sylhet |

**Orders**

| order_id | customer_id |
|---|---|
| 101 | 1 |
| 102 | 2 |

**Order_Items**

| order_id | product |
|---|---|
| 101 | Keyboard |
| 101 | Mouse |
| 102 | Monitor |

Now the data is much cleaner:

- customer data is stored once
- order data is stored once
- product entries are stored in a separate relation

This is the basic idea of normalization: break one messy table into multiple related tables so each table stores one kind of fact.

---

## 7. Final Understanding

Normalization is used to design clean relational databases.

Its goal is to reduce repeated data and avoid data anomalies.

### In one line

Normalization means organizing tables so that data is stored efficiently, consistently, and without unnecessary duplication.

### Quick summary

- Use normalization when consistency matters.
- It reduces redundancy and anomalies.
- It increases the number of tables and joins.
- It is most useful in transactional systems.
- In practice, most systems aim for 1NF, 2NF, and 3NF.

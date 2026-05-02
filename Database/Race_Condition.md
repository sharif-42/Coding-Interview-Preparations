# Race Condition in Database

A race condition happens when two or more transactions or processes access and modify the same data at nearly the same time, and the final result depends on the order in which those operations happen. In simple terms, a race condition means concurrent operations are racing with each other, and if coordination is missing, the data can become incorrect.

Race conditions are very important because they are directly related to:

- concurrency
- transactions
- isolation
- locking
- data consistency

If concurrent transactions read and write the same data without proper coordination, they may produce an incorrect result.

---

## Why Race Conditions Happen

In a large application accessed by thousands of users, concurrency is inevitable. Your application should be able to handle multiple requests simultaneously.

When you execute operations concurrently, the results can be conflicting. For e.g. if you are reading a row while someone else is writing to it simultaneously, then you are bound to get inconsistent data.

If we execute these transactions sequentially, then we don’t need concurrency control. But sequential execution affects scaling of systems. Hence, we cannot avoid concurrent transactions.

But when operations are executed concurrently, how do we make sure that the results are consistent? Race conditions happen because multiple users or systems often try to access the same data at the same time. Common scenarios:

- two users buy the last product at the same time
- two transactions update the same bank account balance
- two users book the same seat
- one transaction reads data while another transaction is modifying it

Without concurrency control, both transactions may act on stale or conflicting data.

---

## Simple Example of a Race Condition

Suppose a product table contains:

| product_id | product_name | stock |
|---|---|---|
| 1 | Keyboard | 1 |

Now two users place an order at the same time.

**Transaction A**

1. reads stock = 1
2. decides item is available

**Transaction B**

1. reads stock = 1
2. also decides item is available

Now both transactions reduce stock.

Possible result:

- both orders succeed
- stock becomes wrong
- the business oversells one item

This is a classic race condition.

---

## Why Race Conditions Are Dangerous?

Race conditions can cause:

- incorrect balances
- oversold inventory
- duplicate bookings
- inconsistent reports
- broken business rules
- hard-to-reproduce production bugs

The dangerous part is that race conditions may not happen every time. They often appear only under load or unlucky timing. That makes them difficult to debug.

Two transactions read the same value and both update it. One update overwrites the other. Example:

- balance = 100
- transaction A adds 20 and writes 120
- transaction B adds 30 and writes 130

Correct answer should be 150, but one update is lost.


## How Relational Databases Handle Race Conditions

Relational databases handle race conditions using concurrency control mechanisms.

The main tools are:

- transactions
- isolation levels
- locks
- MVCC (Multi-Version Concurrency Control) in many databases
- constraints
- atomic update statements

The goal is to ensure correctness even when many users access the same data concurrently.

---

## Now lets discuss few solutions.

## <span style="color:green">Transaction Isolation</span>
ACID-compliant databases need to make sure that each transaction is carried out in isolation. This means that the results of the transaction is only visible after a commit to the database happens. Other processes should not be aware of what’s going on with the records while the transaction is carried out.

What happens when a transaction tries to read a row updated by another transaction?

It depends on what isolation level the database is operating on w.r.t. that particular transaction. Let’s explore the problems that can occur. 
Isolation controls how concurrent transactions interact.

Higher isolation reduces the chance of race conditions, but may reduce performance and concurrency.

### <span style="color:yellow">Problems when transaction isolation is not done</span>

### <span style="color:red">Dirty Read</span>
Let’s take a situation where one transaction updates a row or a table but does not commit the changes. If the database lets another transaction read those changes (before it’s committed) then it’s called a dirty read. Why? Let’s say the first transaction rolls back its changes. The other transaction which read the row/table has stale data. This happens in concurrent systems where multiple transactions are going on in parallel. But this can be prevented by the database and we will explore how later.
```sql
UPDATE balance = 500  -- not commited

SELECT balance → 500 (wrong)
-- if rolled back, wrong data will be returned.
```

### <span style="color:red">Non-repeatable read</span>
Another side effect of concurrent execution of transactions is that consecutive reads can retrieve different results if you allow another transaction to do updates in between. So, if a transaction is querying a row twice, but between the reads, there is another transaction updating the same row, the reads will give different results.
```sql
SELECT balance → 1000

UPDATE balance = 800
COMMIT;

SELECT balance → 800
-- Same query, different result
```

### <span style="color:red">Phantom read</span>
In a similar situation as above, if one transaction does two reads of the same query, but another transaction inserts or deletes new rows leading to a change in the number of rows retrieved by the first transaction in its second read, it’s called a Phantom read. This is similar to a non-repeatable read. The only difference is that while in a non-repeatable read, there will be inconsistency in the values of a row, in phantom reads, the number of rows retrieved by the queries will be different.

```sql
SELECT * FROM orders WHERE amount > 100;
-- 5 rows 

INSERT new order (amount = 200)
COMMIT;

SELECT * FROM orders WHERE amount > 100;
-- 6 rows 😐
```

### <span style="color:red">Write skew</span>

Two concurrent transactions each make a decision based on the same old snapshot and together break a business rule.

This is more advanced, but it often appears in interview discussions about serializable isolation.

---

### How do databases deal with this?
The solution to this is having different levels of isolation. Let’s discuss the most common ones. These are mentioned in increasing order of isolation levels.

## <span style="color:yellow">Levels of Isolation</span>
Here are some level of isolation which help to reduce race condition.

### <span style="color:green">Read uncommitted</span>
This level of isolation lets other transactions read data that was not committed to the database by other transactions. There is no isolation happening here. So, if transaction 1 performs an update and before it’s able to commit, if transaction 2 tries to access the updated data, it will see the new data. This does not solve any issues mentioned above.


### <span style="color:green">Read committed</span>
This, as the name suggests, lets other transactions only read data that is committed to the database. While this looks like an ideal level of isolation, it only solves the dirty read problem mentioned above. If a transaction is updating a row, and another transaction tries to access it, it won’t be able to. But this can still cause non-repeatable and phantom reads because this applies only to updates and not read queries.

### <span style="color:green">Repeatable read</span>
To counter the transactions from getting inconsistent data, we need a higher level of isolation and that is offered by repeatable read. In this, the resource is locked throughout the transaction. So, if the transaction contains two select queries and in between, if another transaction tries to update the same rows, it would be blocked from doing so. This isolation level is not immune to phantom reads though it helps against non-repeatable reads. This is the default level of isolation in many databases.

### <span style="color:green">Serializable</span>
Strongest isolation. Serializable ensures transactions behave as if executed one after another/serially, preventing all concurrency anomalies at the cost of performance.This is the **safest level** against race conditions, but it may reduce throughput and increase waiting or transaction retries.

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- transaction A
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM accounts WHERE balance > 1000;

-- transaction B
BEGIN;
INSERT INTO accounts(balance) VALUES (1500);
COMMIT;

-- if Transaction A wants to commit, PostgreSQL detect a phantom issue and Transaction A will fail ❌
```

---

## <span style="color:green">Locking in Relational Databases</span>

There are various techniques for concurrency control. And one of them is using locks 🔐, Locking is one of the main ways a relational database prevents conflicting concurrent access. It ensures

- Data consistency
- Isolation
- Safe read/write

So, how do they work? Locks work by letting only one transaction modify or access a certain row or table. This makes it look like the transactions are executed one after the other and hence maintaining data integrity. When one transaction locks data, other transactions may need to wait, fail, or read an older version depending on the database and isolation level.


---

## Different types of locks
There are majorly 2 types of locks; 

### <span style="color:yellow">Row-Level Lock</span>
A row-level lock in PostgreSQL locks individual rows returned by a query, preventing other transactions from modifying or acquiring conflicting locks on those rows until the transaction completes. We mostly use this when we build the backend. 

![](/Database/images/race_condition_1.png)

1. <span style="color:green">**FOR UPDATE**</span>: Acquires an exclusive lock on selected rows, preventing other transactions from updating, deleting, or acquiring similar locks on those rows. strongest row-level lock, blocks UPDATE, DELETE, SELECT ... FOR UPDATE
    ```sql
    SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
    ```

2. <span style="color:green">**FOR NO KEY UPDATE**</span>: Acquires a lock that prevents updates which would modify the row’s key (e.g., primary key or indexed columns), but allows some non-key updates. weaker than FOR UPDATE, allows limited concurrent updates

3. <span style="color:green">**FOR SHARE**</span>: Acquires a shared lock on selected rows, allowing other transactions to read the rows but preventing updates or deletes. multiple transactions can hold it. blocks UPDATE, DELETE
    ```sql
    SELECT * FROM accounts WHERE id = 1 FOR SHARE;
    ```

4. <span style="color:green">**FOR KEY SHARE**</span>: Acquires a shared lock that protects foreign key relationships by preventing changes to key columns. weakest row lock, mainly used internally for FK checks


### <span style="color:yellow">Table-Level Locks</span>
Table-level locks are locks applied to an entire table instead of individual rows. When a table-level lock is acquired, it can restrict or block other transactions from reading or writing to that table, depending on the lock mode. In simple terms: **Instead of locking specific rows, the whole table is locked to control access.**

Why Table-Level Locks Are Used
- Schema changes (ALTER, DROP)
- Bulk operations
- Ensuring strict consistency across the entire table
- Preventing concurrent structural or large-scale data modifications

⚠️ Important Notes
- Table-level locks are heavier than row-level locks
- Can significantly reduce concurrency
- Should be used carefully in high-traffic systems

![](/Database/images/race_condition_2.png)

Common Table-Level Locks
1. <span style="color:green">**ACCESS SHARE**</span>
    - Acquired by SELECT
    - Allows multiple reads
    - Does not block writes (in most cases)

2. <span style="color:green">**ROW EXCLUSIVE**</span>
    - Used by INSERT, UPDATE, DELETE
    - Allows reads
    - Restricts conflicting writes

3. <span style="color:green">**ACCESS EXCLUSIVE**</span>
    - Blocks everything:
        - SELECT ❌
        - INSERT ❌
        - UPDATE ❌
    - Used for schema-level operations


## <span style="color:yellow">Pessimistic Locking vs Optimistic Locking</span>

### Pessimistic locking

Pessimistic locking is a concurrency control strategy where a transaction assumes that conflicts are likely to happen, and therefore it **locks the data before reading or modifying it**, preventing other transactions from accessing it concurrently. Example idea:

- read row for update
- hold lock
- modify safely
- commit

This is common in relational databases when using statements like `SELECT ... FOR UPDATE`.

```sql
BEGIN;

SELECT stock
FROM products
WHERE id = 1
FOR UPDATE;

-- row locked

UPDATE products
SET stock = stock - 1
WHERE id = 1;

COMMIT;
```

### Optimistic locking
Optimistic locking is a concurrency control strategy where a transaction does not lock data upfront, assuming conflicts are rare. Instead, it checks for conflicts at update time and proceeds only if the data has not changed.

This is usually done with:

- version number
- timestamp
- last-updated value comparison

Example idea:

1. read row with version = 5
2. update only if version is still 5
3. if not, retry

Optimistic locking is often implemented at the application level, although it works with relational databases. Example

```sql
UPDATE products
SET stock = stock - 1
WHERE id = 1 AND stock > 0;

-- Then check:
-- rows_affected == 1 → success
-- rows_affected == 0 → conflict / no stock

-- No oversell, No blocking
```
Using version number
```sql
UPDATE products
SET stock = stock - 1,
    version = version + 1
WHERE id = 1 AND version = 5;
```

🔥 Optimistic vs Pessimistic
| Feature           | Optimistic | Pessimistic     |
| ----------------- | ---------- | --------------- |
| Lock              | No         | Yes             |
| Blocking          | No         | Yes             |
| Speed             | Fast       | Slower          |
| Safety            | Medium     | High            |
| Conflict handling | Retry      | Prevent upfront |


### Interview point

Relational databases commonly use pessimistic locking internally, but many real applications also use optimistic locking for business workflows.

---

## Practical SQL Example with Locking

Suppose we want to safely reserve a product if stock is available.

### Unsafe approach

```sql
SELECT stock FROM products WHERE product_id = 1;

-- application checks stock > 0

UPDATE products
SET stock = stock - 1
WHERE product_id = 1;
```

This is unsafe because another transaction may change the row between the `SELECT` and the `UPDATE`.

### Safer approach using row lock

```sql
BEGIN;

SELECT stock
FROM products
WHERE product_id = 1
FOR UPDATE;

UPDATE products
SET stock = stock - 1
WHERE product_id = 1
	AND stock > 0;

COMMIT;
```

### What happens here

- `FOR UPDATE` asks the database to lock the selected row for update
- while one transaction holds that lock, another conflicting transaction must wait or fail depending on the database behavior
- this prevents both transactions from reducing the same stock blindly

### Even better pattern

In many cases, the best design is to do the change atomically in one statement:

```sql
UPDATE products
SET stock = stock - 1
WHERE product_id = 1
	AND stock > 0;
```

Then check whether one row was updated.

Why this is good:

- fewer round trips
- smaller race window
- simpler logic

---

## Example: Lost Update in Bank Balance

Suppose balance is 100.

Two transactions run at the same time.

### Transaction A

1. reads balance = 100
2. calculates new balance = 120
3. writes 120

### Transaction B

1. reads balance = 100
2. calculates new balance = 130
3. writes 130

Final balance becomes 130.

But the correct result should be 150.

This is a lost update race condition.

### Safer SQL approach

Instead of:

```sql
SELECT balance FROM accounts WHERE id = 1;
UPDATE accounts SET balance = 120 WHERE id = 1;
```

Use:

```sql
UPDATE accounts
SET balance = balance + 20
WHERE id = 1;
```

This keeps the change atomic at the database level.

---

## How MVCC Helps

Many relational databases such as PostgreSQL use MVCC.

MVCC means Multi-Version Concurrency Control.

Instead of blocking every reader and writer, the database keeps multiple versions of rows so reads and writes can happen more efficiently.

Benefits:

- readers often do not block writers
- writers often do not block readers in the same way as simple locking systems
- concurrency is improved

Important point:

MVCC does not remove the need for isolation rules or locks. It is one implemNoentation technique for concurrency control.

---

## Problems Locking Can Create

Locking solves race conditions, but it can introduce its own issues.

### Blocking

One transaction waits because another transaction holds a lock.

### Deadlock

Transaction A waits for a lock held by transaction B, while transaction B waits for a lock held by transaction A. The database detects this and usually aborts one transaction.

![deadlock](/Database/images/race_condition_3_deadlock.png)


### Reduced concurrency

Very aggressive locking can make the system slower because too many transactions are waiting.

### Interview point

Correctness and performance must be balanced. More locking usually means more safety, but less concurrency.

---

## Best Practices to Avoid Race Conditions

- use transactions for related operations
- choose the correct isolation level
- use `SELECT ... FOR UPDATE` when you truly need pessimistic locking
- prefer atomic SQL statements like `SET count = count - 1`
- enforce constraints in the database
- keep transactions short
- access rows in a consistent order to reduce deadlocks
- retry transactions when serialization failure or deadlock occurs
- use optimistic locking when appropriate

---

## 17. Interview Questions and Short Answers

### What is a race condition?

A race condition is a concurrency issue where the final result depends on the timing of concurrent operations on shared data.

### Why do race conditions happen in databases?

They happen when multiple transactions access and modify the same data at the same time without proper coordination.

### How do relational databases handle race conditions?

By using transactions, isolation levels, locking, MVCC, and constraints.

### What is locking?

Locking is a concurrency control mechanism that restricts simultaneous conflicting access to database resources.

### What is the difference between shared lock and exclusive lock?

- shared lock is mainly for safe reading
- exclusive lock is for writing and prevents conflicting access

### What is `SELECT ... FOR UPDATE`?

It reads rows and locks them for update so other conflicting transactions cannot modify them until the current transaction finishes.

### What is the difference between optimistic and pessimistic locking?

- pessimistic locking locks early because conflict is expected
- optimistic locking checks for conflict at update time and retries if needed

### Is locking the only way to solve race conditions?

No. Atomic statements, isolation levels, MVCC, constraints, and application-level retry logic also help.

---

## 18. Final Understanding

Race conditions happen when concurrent operations on the same data interfere with each other and produce incorrect results.

Relational databases handle them through concurrency control techniques such as:

- transactions
- isolation levels
- row and table locking
- MVCC
- constraints
- atomic updates

### In one line

A race condition is a timing-related concurrency bug, and relational databases reduce it through transactions, locking, and isolation control.

### Quick summary

- race conditions are caused by concurrent access to shared data
- they can cause lost updates, overselling, and inconsistent state
- relational databases handle them with transactions and concurrency control
- locking is a key tool, but too much locking reduces concurrency
- atomic SQL statements are often safer than read-then-write logic

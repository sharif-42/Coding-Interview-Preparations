# ACID Properties of Database

ACID is a set of properties that makes database transactions reliable.

ACID stands for:

- Atomicity
- Consistency
- Isolation
- Durability

These properties are mainly discussed in relational databases such as PostgreSQL, MySQL, Oracle, and SQL Server, but the ideas are important in distributed systems as well.

---

## 1. What Is a Transaction?

A transaction is a group of one or more database operations treated as one logical unit of work.

Examples:

- transferring money from one bank account to another
- placing an order and reducing product stock
- booking a seat and generating a payment record

In such cases, all related operations must behave correctly together. That is exactly why ACID exists.

### Simple example

Suppose Alice sends 100 dollars to Bob.

This usually means:

1. subtract 100 from Alice's account
2. add 100 to Bob's account

If only the first step happens and the second step fails, the data becomes wrong.

ACID ensures this kind of transaction is handled safely.

---

## 2. Formal Definitions of ACID

### Atomicity

**Formal definition:** A transaction is atomic if it is treated as an indivisible unit, meaning either all of its operations are committed or none of them are applied.

In simple terms, atomicity means all or nothing.

If one step fails, the whole transaction is rolled back.

### Consistency

**Formal definition:** A transaction is consistent if it takes the database from one valid state to another valid state while preserving all defined rules, constraints, and invariants.

In simple terms, consistency means the database must remain logically correct after the transaction.

Examples of rules:

- primary key must be unique
- foreign key must refer to an existing row
- account balance cannot go below allowed limits
- total quantity should not become negative

### Isolation

**Formal definition:** A transaction is isolated if concurrent transactions do not interfere with each other in a way that produces incorrect results, and the outcome is equivalent to some valid serial execution.

In simple terms, isolation means concurrent transactions should behave as if they ran one after another.

### Durability

**Formal definition:** A transaction is durable if, once committed, its effects are permanently stored and survive crashes, power failure, or system restart.

In simple terms, durability means committed data is not lost.

---

## 3. What ACID Means in Practice

When a database says it supports ACID transactions, it usually means:

- changes can be committed or rolled back safely
- concurrent transactions are managed carefully
- committed data survives failures
- constraints and integrity rules are enforced

This is especially important in systems where correctness matters more than raw speed.

Examples:

- banking
- payment processing
- e-commerce orders
- inventory systems
- airline or hotel booking systems
- payroll systems

---

## 4. Why We Need ACID

We need ACID because real systems face failures and concurrency all the time.

Without ACID, we can easily get:

- partial updates
- duplicate or missing data
- incorrect balances
- oversold inventory
- corrupted relationships between tables
- lost committed changes after a crash

### Example without ACID

Imagine an online store where two users buy the last item at the same time.

Without proper transaction handling:

- both users may read stock as 1
- both may place the order
- final stock may become incorrect

ACID, especially isolation and consistency, helps prevent that.

### Example with a crash

Suppose payment is marked successful, but order creation is not saved because the server crashes in the middle.

Without atomicity and durability:

- customer money may be charged
- but order may not exist in the system

That creates a serious business problem.

---

## 5. Detailed Explanation of Each Property

### 5.1 Atomicity

Atomicity guarantees that a transaction is completed fully or not at all.

#### Banking example

Transaction:

1. deduct 100 from Alice
2. add 100 to Bob

If the system crashes after step 1 but before step 2, the transaction must be rolled back.

So either:

- both changes happen, or
- neither change happens

#### How databases support atomicity

Databases use mechanisms such as:

- transaction logs
- undo logs
- rollback operations

#### Interview point

Atomicity does not mean one SQL statement only. A transaction may contain many statements, but they succeed or fail as one unit.

---

### 5.2 Consistency

Consistency ensures that database rules remain valid before and after a transaction.

#### Example

Suppose a table has a foreign key from `orders.customer_id` to `customers.id`.

If you insert an order for a customer that does not exist, the database should reject it.

That preserves consistency.

#### Important clarification

Consistency in ACID does not simply mean that two users see the same data.

It means the transaction must preserve:

- constraints
- triggers
- cascades
- business rules
- valid state transitions

#### Interview point

Many candidates confuse consistency with isolation. They are different:

- consistency is about correctness of rules
- isolation is about concurrent execution behavior

---

### 5.3 Isolation

Isolation ensures concurrent transactions do not break correctness.

If many users access the database at the same time, one transaction should not improperly affect another.

#### Problem example

Two transactions read the same balance of 1000 and both try to withdraw 800.

Without proper isolation, both may succeed, which can produce invalid data.

#### Isolation idea

The ideal result is the same as if transactions ran serially, one after another.

#### Common read anomalies

Isolation is often explained using these anomalies:

- **Dirty read:** reading uncommitted data from another transaction
- **Non-repeatable read:** reading the same row twice and getting different values
- **Phantom read:** re-running a query and seeing extra or missing rows due to another committed transaction

#### Interview point

Isolation is not free. Stronger isolation usually gives better correctness but can reduce concurrency and performance.

---

### 5.4 Durability

Durability ensures that once a transaction is committed, the result is permanent.

Even if the database crashes immediately after commit, the data should still be there after recovery.

#### How durability is supported

Databases usually achieve this using:

- write-ahead logs
- redo logs
- checkpoints
- disk flush mechanisms
- replication in some systems

#### Interview point

`COMMIT` is the key moment. Before commit, changes may still be rolled back. After commit, changes must survive failures.

---

## 6. ACID Example Using Bank Transfer

Suppose the transaction is:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

How ACID applies here:

- **Atomicity:** both updates happen together or both are rolled back
- **Consistency:** total money remains correct if business rules are preserved
- **Isolation:** other concurrent transactions should not see broken intermediate state
- **Durability:** once committed, the transfer survives a crash

---

## 7. Isolation Levels and Their Importance

In interviews, ACID often leads to a discussion of isolation levels.

Common SQL isolation levels are:

- Read Uncommitted
- Read Committed
- Repeatable Read
- Serializable

### Read Uncommitted

Lowest isolation. Dirty reads may happen.

### Read Committed

Prevents dirty reads, but non-repeatable reads can still happen.

### Repeatable Read

Prevents dirty reads and non-repeatable reads, but phantom reads may still be discussed depending on database implementation.

### Serializable

Highest isolation. Transactions behave closest to serial execution, but performance cost is highest.

### Important interview point

ACID does not mean every transaction always runs at the highest isolation level.

Real systems choose the isolation level based on the balance between correctness and performance.

---

## 8. How Databases Implement ACID

Different databases implement ACID using internal techniques such as:

- locks
- MVCC (Multi-Version Concurrency Control)
- undo and redo logs
- write-ahead logging
- checkpoints and crash recovery

### Short explanation

- **Locks** control concurrent access
- **MVCC** allows readers and writers to work with multiple versions of data
- **Logs** help undo incomplete transactions and redo committed ones after failure

You usually do not need to memorize every internal detail for interviews, but you should know the general idea.

---

## 9. ACID vs BASE

This comparison is common in system design interviews.

### ACID

- strong correctness focus
- common in relational databases
- best for transactions where consistency matters a lot

### BASE

BASE stands for:

- Basically Available
- Soft state
- Eventual consistency

BASE is common in some NoSQL and distributed systems where high availability and scalability are prioritized.

### Key idea

ACID prioritizes transaction correctness.

BASE accepts temporary inconsistency in exchange for scale or availability.

This does not mean NoSQL databases never support ACID. Many modern databases support ACID for at least some operations.

---

## 10. Common Interview Questions and Answers

### What does ACID stand for?

Atomicity, Consistency, Isolation, Durability.

### Why is ACID important?

It ensures reliable transactions, protects data correctness, and prevents corruption during failures or concurrent access.

### What is the difference between atomicity and consistency?

- atomicity means all-or-nothing execution
- consistency means the database remains valid according to rules and constraints

### What is the difference between consistency and isolation?

- consistency is about valid data state
- isolation is about how concurrent transactions interact

### Can a database be ACID and still have performance issues?

Yes. Stronger guarantees, especially stronger isolation, can reduce throughput and increase locking or waiting.

### Is ACID only for SQL databases?

No. ACID is strongly associated with relational databases, but non-relational databases can also support ACID transactions depending on the product.

### Does COMMIT guarantee durability?

Yes, in an ACID-compliant system, committed changes must survive crashes, subject to the database's actual guarantees and configuration.

### Is consistency checked only by the database?

Not always. Some consistency rules are enforced by constraints in the database, while others come from application-level business logic.

---

## 11. Common Misconceptions

### Misconception 1: Consistency means same data for everyone at all times

That idea is closer to distributed system consistency discussions. In ACID, consistency means preserving rules and valid states.

### Misconception 2: Isolation means transactions never run in parallel

Transactions can run concurrently. Isolation means concurrency is controlled so results stay correct.

### Misconception 3: Durability means data can never be lost under any condition

Durability means committed data survives normal crash scenarios according to the database system design. Catastrophic storage failure without backup is a separate issue.

### Misconception 4: Atomicity and rollback are the same thing

Rollback is one mechanism used to preserve atomicity. Atomicity is the property; rollback is part of the implementation.

---

## 12. When ACID Is Most Important

ACID is especially important in:

- banking and finance
- order and payment systems
- reservation systems
- stock and inventory management
- healthcare records
- payroll and accounting

In these systems, wrong data is much more expensive than slightly slower performance.

---

## 13. Final Understanding

ACID makes transactions safe and dependable.

It ensures that:

- a transaction is fully completed or fully rejected
- data remains valid
- concurrent transactions do not corrupt each other
- committed changes are not lost after failure

### In one line

ACID is the set of properties that guarantees reliable and correct transaction processing in a database.

### Quick summary

- **Atomicity:** all or nothing
- **Consistency:** valid state to valid state
- **Isolation:** safe concurrency
- **Durability:** committed data is permanent

If you understand these four properties clearly, most basic and intermediate database interview questions on transactions become much easier.

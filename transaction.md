# what is different between non_repeatable read and phantom read?

**Non-repeatable reads** and **phantom reads** are both phenomena that can occur in database systems, 
particularly in transactional environments, due to concurrent transactions accessing the same data. 
However, they represent slightly different scenarios:

- **Non-repeatable Read:**
    - A non-repeatable read occurs when a transaction reads the same row multiple times 
        within a single transaction, but gets different results each time.
    - This inconsistency happens because another transaction modifies or deletes the row 
        between the reads of the first transaction.
    - In other words, a non-repeatable read means that the data changes (due to updates or deletes) 
        between the reads within a single transaction, causing the transaction to get different values for the same row.


Let's demonstrate a non-repeatable read scenario in PostgreSQL. Suppose we have a table named accounts with the following structure:

```sql
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    account_number VARCHAR(20) UNIQUE,
    balance INTEGER
);

INSERT INTO accounts (account_number, balance) VALUES ('1001', 500);
```

Now, let's consider two transactions running concurrently:

```sql
-- Transaction 1:

BEGIN;
SELECT * FROM accounts WHERE account_number = '1001';
-- Other operations...
COMMIT;

```

```sql
-- Transaction 2:
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE account_number = '1001';
-- Other operations...
COMMIT;
```

Here's how a non-repeatable read scenario can occur:

1) **Versioning of Data:**
    - Initially, Transaction 1 starts and reads the balance of the account with account number '1001'.
    - Let's say the balance read by Transaction 1 is 500.
2) **Visibility Rules:**
    - Meanwhile, Transaction 2 starts and updates the balance of the account with account number '1001' by adding 100 to it.
    - The balance is now updated to 600.
3) **Transaction Isolation:**
    - Since Transaction 1 is running at the default Read Committed isolation level, it only sees committed data.
    - However, it doesn't prevent other transactions from modifying data while it's running.
3) **Non-Repeatable Read:**
    - Now, if Transaction 1 tries to read the balance of the account with account number '1001' again, it may get a different value.
    - It might read the updated balance of 600 instead of the previously read 500, leading to a non-repeatable read scenario.

This demonstrates how a non-repeatable read can occur when one transaction reads the same data multiple times within its scope, but the data changes between the reads due to other concurrent transactions modifying it.

- **Phantom Read:**
    - A phantom read occurs when a transaction re-executes a query, and the set of rows returned by the 
        query changes between executions due to another transaction committing new data that matches the criteria of the query.
    - Unlike non-repeatable reads, phantom reads involve the addition of new rows that satisfy a search 
        condition rather than changes to existing rows.
    - For example, if a transaction reads all rows where a certain condition is met, and in between 
        subsequent reads, another transaction inserts a new row that meets the condition, the first 
        transaction would see the new row in the subsequent read, causing a phantom read.

Let's illustrate a phantom read scenario in PostgreSQL. Consider a table named orders with the following structure:

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    order_number VARCHAR(20) UNIQUE,
    total_amount INTEGER
);

INSERT INTO orders (order_number, total_amount) VALUES ('1001', 100);
```

Now, let's consider two transactions running concurrently:

```sql
-- Transaction 1:
BEGIN;
SELECT * FROM orders WHERE total_amount > 90;
-- Other operations...
COMMIT;

```

```sql
-- Transaction 2:
BEGIN;
INSERT INTO orders (order_number, total_amount) VALUES ('1002', 120);
-- Other operations...
COMMIT;
```

Here's how a phantom read scenario can occur:↳

1) **Versioning of Data:**
    - Initially, Transaction 1 starts and reads orders with a total amount greater than 90.
    - It retrieves the order with order number '1001', which has a total amount of 100.
2) **Visibility Rules:**
    - Meanwhile, Transaction 2 starts and inserts a new order with order number '1002' and a total amount of 120.
3) **Transaction Isolation:**
    - Since Transaction 1 is running at the default Read Committed isolation level, it only sees committed data.
    - However, it doesn't prevent other transactions from inserting new data while it's running.
4) **Phantom Read:**
    - Now, if Transaction 1 re-executes the same query to retrieve orders with a total amount greater than 90, 
        it might encounter a different result set.
    - It might now retrieve both orders '1001' and '1002', even though the new order '1002'
        did not exist when Transaction 1 initially executed the query.
    - This is a phantom read scenario, where the result set of a query changes between consecutive 
        executions due to new data inserted by other transactions.

This demonstrates how a phantom read can occur when a transaction re-executes a query, and the result set changes between executions due to new data inserted by concurrent transactions.

## In summary

- **Non-repeatable** reads involve changes to existing rows, where the same row is read multiple 
    times within a transaction, but its value changes due to modifications or deletions by other transactions.

- **Phantom** reads involve the addition of new rows that meet the criteria of a 
    query between consecutive executions of the same query within a transaction,

Both phenomena highlight potential concurrency issues in database systems and are typically managed using isolation levels in transactions to control the visibility of changes made by concurrent transaction


# Isolation levels


PostgreSQL, like many other relational database management systems (RDBMS), provides different 
isolation levels to control how transactions interact with each other and the data they access. 
Isolation levels determine the visibility of changes made by one transaction to other transactions 
running concurrently. PostgreSQL supports the following isolation levels, each offering 
different trade-offs between consistency, concurrency, and performance:

- **Read Uncommitted:**
    - In the Read Uncommitted isolation level, transactions can see uncommitted changes made by other transactions.
    - This level offers the highest concurrency but sacrifices consistency, as transactions can read data that might be rolled back later.
- **Read Committed:**
    - Read Committed ensures that transactions only see changes committed by other transactions.
    - Transactions do not see changes made by other transactions until they are committed.
    - This level prevents dirty reads but allows non-repeatable reads and phantom reads.

- **Repeatable Read:**
    - Repeatable Read ensures that a transaction sees a consistent snapshot of the database at the start of the transaction.
    - Transactions do not see changes made by other transactions after the transaction begins.
    - This level prevents non-repeatable reads but allows phantom reads.
- **Serializable:**
    - Serializable provides the highest level of isolation by ensuring that transactions 
        execute as if they were the only transactions running on the database.
    - It prevents all anomalies, including dirty reads, non-repeatable reads, and phantom reads.
    - Serializable achieves this by serializing transactions, effectively executing them one after 
        the other, which can lead to reduced concurrency and potential performance impacts.

To set the isolation level for a transaction in PostgreSQL, you can use the `SET TRANSACTION` 
command at the beginning of your transaction block:

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL
```

# MVCC

MVCC stands for Multi-Version Concurrency Control, and it's a key feature of PostgreSQL and many 
other modern relational database management systems. It's a concurrency control method that allows 
multiple transactions to access and modify the same data concurrently while ensuring data consistency and integrity.

Here's how MVCC works in PostgreSQL:

1) **Versioning of Data:**
    - When a row is inserted or updated, PostgreSQL doesn't immediately overwrite the existing row.
    - Instead, it creates a new version of the row, called a tuple or a version, with a new transaction 
        ID and timestamp indicating when the change occurred.
    - The old version of the row is retained in the database and marked as obsolete.
2) **Visibility Rules:**
    - When a transaction reads data, PostgreSQL uses the transaction's start timestamp to 
        determine which versions of rows are visible to the transaction.
    - A transaction only sees rows that were committed before the transaction started and are not marked as obsolete.
    - This means that each transaction sees a consistent snapshot of the database as it existed at the transaction's 
        start time, even if other transactions are concurrently modifying the data.
3) **Concurrency Control:**
    - MVCC provides concurrency control by allowing transactions to proceed without locking rows for read access.
    - Multiple transactions can read the same row concurrently without blocking each other.
    - Transactions that modify data acquire row-level locks to prevent conflicts with other transactions that might be concurrentl
4) **Transaction Isolation:**
    - MVCC allows PostgreSQL to support different transaction isolation levels, such as 
        `Read Committed`, `Repeatable Read`, and `Serializable`, by managing the visibility of data changes based on the isolation level.
5) **Garbage Collection:**
    - Periodically, PostgreSQL performs a process known as vacuuming to reclaim storage space by removing obsolete versions of rows.
    - Vacuuming identifies and removes old row versions that are no longer needed for maintaining data consistency.
    - This ensures that the database doesn't grow indefinitely due to the accumulation of obsolete data versions.

Let's walk through an example of how MVCC works in PostgreSQL with a simple scenario:

Let's consider a table called employees with the following structure:

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    salary INTEGER
);

INSERT INTO employees (name, salary) VALUES ('Alice', 50000);
INSERT INTO employees (name, salary) VALUES ('Bob', 60000);
```

Now, let's consider two transactions running concurrently:
```sql
-- Transaction 1:
BEGIN;
UPDATE employees SET salary = 55000 WHERE name = 'Alice';
-- Other operations...
COMMIT;

```

```sql
-- Transaction 2:
BEGIN;
SELECT * FROM employees;
-- Other operations...
COMMIT;

```

Here's how MVCC works in this scenario:

1) **Versioning of Data:**
    - When Transaction 1 updates Alice's salary, PostgreSQL creates a new version of the row with the 
        updated salary (e.g., (1, 'Alice', 55000)).
    - The old version of the row (1, 'Alice', 50000) is retained in the database.
2) **Visibility Rules:**
    - Transaction 2 starts after Transaction 1 begins.
    - Even though Transaction 1 has updated Alice's salary, Transaction 2 only sees the 
        original version of the row with Alice's salary as 50000.
    - This ensures that Transaction 2 sees a consistent snapshot of the database as it existed at the start of the transaction.
3) **Concurrency Control:**
    - Transaction 1 acquires locks on the rows it modifies (e.g., the row with id=1) to prevent conflicts with other transactions.
    - Transaction 2 can read the rows without blocking, as it does not modify data and only sees committed and visible versions of the rows.
4) **Transaction Isolation:**
    - MVCC ensures that each transaction operates in isolation and sees a consistent view of the database at its start time.
    - This isolation prevents Transaction 2 from seeing the changes made by Transaction 1 until Transaction 1 commits its changes.

After Transaction 1 commits its changes, the updated salary for Alice becomes visible to subsequent transactions. MVCC ensures that even though multiple transactions are accessing and modifying the same data concurrently, they do so in a consistent and isolated manner, preserving data integrity and concurrency.

# ACID        

In PostgreSQL and other database systems, ACID is an acronym that stands for Atomicity,
Consistency, Isolation, and Durability. It represents a set of properties that guarantee the 
reliability and integrity of transactions. Let's break down each component:



- **Atomicity:**
    - Atomicity ensures that a transaction is treated as a single unit of work that either 
        completes successfully in its entirety or is fully rolled back if any part of it fails.
    - In PostgreSQL, each SQL statement within a transaction is atomic. If any statement within 
        the transaction fails, all changes made by previous statements are rolled back, leaving the database in its original state.
- **Consistency:**
    - PostgreSQL enforces consistency through constraints, triggers, and foreign key 
        relationships, ensuring that data modifications maintain the integrity of the database schema.
        - Constraints enforce rules at the database level. Common types include
            - **Constraints:**
                - **Primary Key:** Ensures each row is unique and identifiable.
                - **Foreign Key:** Enforces referential integrity between tables, ensuring a row in a child table references a valid row in a parent table.
                - **Unique:** Ensures that values in a column (or set of columns) are unique across rows.
                - **Check:** Validates data according to a specific condition (e.g., age must be greater than 0).
                - **Not Null:** Ensures that a column cannot contain null values. 
            - **Triggers:**
                - Triggers are user-defined functions that are automatically executed in response to certain events (e.g., INSERT, UPDATE, or DELETE operations).
                - They can enforce complex business logic or data consistency rules that go beyond standard constraints.
                - For example, a trigger can update a modified_at column automatically or validate data across multiple columns or tables. 
            - **Foreign Key Relationships:**        
                - Foreign keys enforce consistency between related tables by ensuring that a value in one table (child) exists in another table (parent).
                - Options like ON DELETE CASCADE or ON UPDATE CASCADE allow automatic deletion or updating of child rows if the corresponding parent row changes, maintaining referential integrity.   

- **Isolation:**
    - Isolation ensures that the execution of multiple transactions concurrently yields the same result as 
        if the transactions were executed sequentially, without interfering with each other.
    - PostgreSQL supports different transaction isolation levels, such as `Read Committed`, `Repeatable Read`, and `Serializable`
    - Isolation is achieved through concurrency control mechanisms like MVCC (Multi-Version Concurrency Control), which allows transactions to access and modify data without blocking each other.
- **Durability:**
    - PostgreSQL achieves durability by writing transaction changes to the transaction log (WAL - Write-Ahead Logging) before modifying data in the main database. This ensures that committed transactions can be replayed and recovered in the event of a crash.

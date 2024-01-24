#  Database Desgin Problem
   ### What issues may arise while booking in current setup?
   1. Same Seat selection by two users at the same time for the same Flight
   2. One seat and two or more users trying to book it at the same time : Concurrency
   3. During payment there may be a chance of failure of payment gateway
   4. If payment is successful but client crashes before booking is confirmed


## Transactions in Database
> For example, if we are transferring money from one account to another, we need to first debit money from one account and then credit it to another account. If any of these two operations fails, the whole transaction fails. In such cases, we need to ensure that either both the operations succeed or fail together. This is where transactions come into picture.

In real life scenario, we might need to execute a series of queries in order to accomplish a single task. 
We might do a club of CRUD operations. These series operations can exectue a single unit of work and hence these series of operations are called Database transactions.

Now during the transaction execution our DB might go through in a lot of changes and can be in an inconsistent state.
In such situation ACID properties of DB comes into picture.

#### States of a Trasaction
 1. Begin : The transaction begins.
 2. Commit : All the changes are applied successfully.
 3. Rollback : Something happened in between and whatever changes were successfully will be reverted.

## ``` ACID stands for Atomicity, Consistency, Isolation, Durability.  ```

### Atomicity
A trasaction is a bundle of statement that intends to achieve one final state. When we are attempting a trasaction, we either want to complete all the statement or none of them. We never want an intermediate state. This is called Atomicity.


### Consistency
Data stored in the DB is always valid in a consitent state. All the constraints are satisfied. For example, if we have a constraint that a particular column cannot be null, then the DB will always ensure that the column is not null.

### Isoaltion [IMP]
It's an ability of multiple transaction to execute without interfering with one another.

Q. Can isolation make the DB slower?
Ans. Yes, because it has to ensure that the transaction are not interfering with each other. But we don't need isolation everytime and hence we can set the isolation level. So we can also have parallel execution of transaction.

### Durability
If something changed in the DB and any unexpected event occurs then our changes should persist. For example, if we have a power failure or system failure then our changes should not be lost.

## Execution Anomalies
1. Read-Write Conflict
2. Wrire-Read Conflict
3. Write-Write Conflict


## How Databases ensure Atomicity?
1. Logging : DBMS logs all actions that it's doing so that later it can undo it.
2. Shadow Paging : DBMS makes copies of actions/pages. This copy is initially considered as a temporary copy. If the transaction is successful then it starts pointing to the new temporary copy.

# Atomicity in MySQL
Most databases prefer logging mysql also use this. this things happens internally so as a user we don't need to worry about this.The built-in transaction management in MySQL is designed to handle atomicity efficiently and reliably. Transaction can have three state: begin,commit or rollback
After each commit or rollback, database remains in a consistent state.
In order to handle rollback there are two types of logs: UNDO and REDO logs.

##### Undo logs : 
This log contains records about how to undo the last change done by a transaction. If any other transaction need the original data as a part of consistent read operation the unmodified data is retrieved from the undo log.

##### Redo logs :
By definition, redo logs is a disk based data structure used for crash recovery to correct data written by incomplete transactions. The changes which could make it upto the data files before the crash or any other reasons are reapplied automatically during restart of the server after a crash.


Refer  : https://en.wikipedia.org/wiki/Isolation_(database_systems)
(also read dirty read, non repeatable read, phantom read)
### ISOLATION LEVELS
(see actually what happened if database apply best isolation level then it will reduce speed or if it will apply no isolation then many concurrency problem occur so it leaves to user that you decide which isolation technique you want you used and gives us method)

#### 1. Read Uncommitted
There is almost no isolation. 
One transaction reads the latest uncommitted value at every step that can be updated from other uncommitted transactions.
So dirty reads, non-repeatable reads and phantom reads are possible.

#### 2. Read Committed
Here dirty reads is avoided, because any uncommited changes are not visible to any other trasactions until we commit.
In this level, each select statement will have it's own snapshot of data which can be problematic if we execute the same select statement again, because some other transaction might commit an update and we will see new data in the second select. (see what happened our transaction t1 has not update something but when we read we get the changed/new value because t2 comes in b/w commit and update--problem non-repeatable read)


#### 3. Repeatable Read
A snapshot of select is taken when it runs first time during a transaction and same snapshot is used throughout the transaction,when same select is executed.
A txn running at this level does not take into account any changes to data made by other transactions. 
But this bring us to phantom reads problem, i.e. a new row can exist in between txn which was not before.
------------------------------------------------------------------------------
see this--
In the context of database transactions, a phantom read occurs when a transaction retrieves a set of rows based on a given query, and a concurrent transaction inserts or deletes rows that meet the query criteria, causing the first transaction to retrieve a different set of rows when it re-executes the same query.

In MySQL, the default isolation level is `REPEATABLE READ`, which is designed to prevent phantom reads. However, there are certain scenarios where phantom reads can still occur. Here's an example:

1. Transaction 1 (`Tx1`) starts and reads from a table.
2. Transaction 2 (`Tx2`) starts, inserts a new row into the table, and commits.
3. `Tx1` reads from the table again. The newly inserted row by `Tx2` is not visible due to the snapshot isolation of `REPEATABLE READ`. So far, there's no phantom read.
4. `Tx1` tries to update the table based on some condition that includes the newly inserted row by `Tx2`. Surprisingly, the update operation affects the new row, even though it was not visible in the previous read operation.
5. Now, if `Tx1` reads from the table again, it sees the newly inserted and updated row. This is a phantom read¹.

This behavior is quite unexpected and counter-intuitive, as the table never had one single row committed; two rows were inserted and committed. This example illustrates how a `REPEATABLE READ` isolation level can lead to a phantom read in MySQL. It's important to note that this is a specific behavior of MySQL's implementation of the `REPEATABLE READ` isolation level.

Let's consider an example of how `REPEATABLE READ` can lead to a phantom read in MySQL:

Let's say we have a table `Orders` with a column `Status` that can be 'Pending', 'Shipped', or 'Delivered'. 

```sql
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY,
    Status VARCHAR(20)
);
```

Now, consider the following sequence of operations:

**Transaction 1:**
```sql
START TRANSACTION;
SELECT * FROM Orders WHERE Status = 'Pending';
```
This returns all orders that are currently pending.

**Transaction 2:**
```sql
START TRANSACTION;
INSERT INTO Orders VALUES (101, 'Pending');
COMMIT;
```
This inserts a new pending order.

**Transaction 1:**
```sql
SELECT * FROM Orders WHERE Status = 'Pending';
```
This returns the same set of rows as the first `SELECT` statement because of the `REPEATABLE READ` isolation level. The newly inserted row by Transaction 2 is not visible yet.

**Transaction 1:**
```sql
UPDATE Orders SET Status = 'Shipped' WHERE Status = 'Pending';
```
This updates all pending orders to 'Shipped', including the new order inserted by Transaction 2.

**Transaction 1:**
```sql
SELECT * FROM Orders WHERE Status = 'Pending';
```
Now, this returns a different set of rows than the previous `SELECT` statements because the `UPDATE` statement affected the new row inserted by Transaction 2. This is a phantom read.

**Transaction 1:**
```sql
COMMIT;
```
This commits the transaction.

In this example, even though the `REPEATABLE READ` isolation level was used, Transaction 1 experienced a phantom read because the `UPDATE` statement affected a row that wasn't visible in the `SELECT` statements. This is a specific behavior of MySQL's implementation of the `REPEATABLE READ` isolation level.
------------------------------------------------------------------------------


#### 4. Serializable
It completely isolates the effect of one txn from others. 
It is a repeatable read with more isolations to avoid phantom reads.

------------------------------------------------------------------------------
see ex-i am searching some name in instagram and at the same time a user is creating his account with the same name so here concurrency is not a very big problem but in banking we are talking about money it's a very big problem.

------------------------------------------------------------------------------

# Durability in MySQL
The DB should be durable enough to hold all the latest updates even if system fails or restarts. 
If a txn updates a chunk of data in DB and commits. The DB will hold the new data. If txn commits but system fails before data could be written then data should be written back when system restarts.

> MySQL storage engine : InnoDB(default engine)

# Consistency in MySQL
Consistency in InnoDB involves protecting data from crashes and maintaining data integrity and consitency.
Two important features of InnoDB are:
1. Double write buffer
2. Crash Recovery--(using undo&redo logs)

### Page  : 
It's a unit that specify how much data can be transferred between disk and memory(os concepts). A page can contain one or more rows. If one row doesn't fit in the page InnoDB sets up additional pointers style data structures so that whole info of one row can go in a page.

### Flush :
When we write something to the database, it is not written instantly for performance reasons in MySQL. It instead stores that either in memory or in a temporary disk storage space. InnoDB storage structure that are periodically flushed include redo logs, undo logs and buffer pool.
Flushing can happen because a memory area became full and system needs to free some space because if there is a commit involes then txn has to be finalized.

## Double write buffer
It is a storage area where InnoDB writes pages flushed from buffer pool before writing the pages to their position in data files. If a system crashes in middle of a page write, InnoDB can find a good copy from double write buffer.


#  Race Condition
## Locking Mechanism

1. Shared Locks :
It allow multiple txn to read data at the same time but restrict any of them from writing.

2. Exclusive Locks :
This prevents txn from reading or writing data at the same point of time.

3. Intention Locks :
This is use to specify a txn is planning to read or write a certain section of data.

4. Row level Locks :
This allows txn to lock only a specific row.


MySQL : MVCC (Multi Version Concurrency Control) type database
It allow multiple txn to read or write same data without much conflict.

Every txn in MySQL sort of capture the data it is about to modify at start of txn and write the change to an entirely different version of data. This is called MVCC.

------------------------------------------------------------------------------
how we can achieve acid?

atomicity-->using transaction

consistency-->total no of seat remains same=sold+unsold
              using constraint of foreign key(on delete cascade,on update cascade)
              make isolation level serializable
              
isolation-->by default(Repeatable Read but still have phamton read problem)
            can make isolation level serializable

durability--> by default logs maintain if system crashes 

what if two users tried to book tickets at same time?
using indexing

problem---
1.The same user clicks on the “book” button multiple times. 
idempotent api

2.Multiple users try to book the same Seat/Room/Slot at the same time.
(refer https://medium.com/@abhishekranjandev/concurrency-conundrum-in-booking-systems-2e53dc717e8c)
The simplest way to solve the above problem is: Database Locking (Optimistic and Pessimistic)
optimistic concurrency control--> 
Optimistic Locking is a strategy where you read a record, take note of a version number and check that the version hasn’t changed before you write the record back. checking for conflict before commiting.

pessimistic concurrency control--> 
using locks

--20 min window slot just like flipkart flights

------------------------------------------------------------------------------
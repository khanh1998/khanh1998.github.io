---
date: '2022-01-02T20:24:18+07:00'
draft: false
title: 'Isolation Level and Read Phenomena'
tags: ['database']
---
## 1. Read phenomena
When transaction A reads the data that might be changed by transaction B.
### 1.1 Dirty reads:
Is when a transaction read uncommitted data from another transaction. Example:
|**Transaction A**|**Transaction B**|
| :-------------- | --------------: |
| begin transaction;||
| select age from employee where id = 1;||
| // age = 24||
||begin transaction;|
||update employee set age = 25 where id = 1;|
||// age = 25|
|select age from employee where id = 1;||
|// age = 25||
||rollback;|
|select age from employee where id = 1;||
|// age = 24||
|commit;||

### 1.2 Non-repeatable reads
When during a transaction, you retrieve a row two times, and the second time, you got a slightly different row. It is different from the dirty read in that this time, it read committed data. 

|**Transaction A**|**Transaction B**|
|:----------------|----------------:|
|begin transaction;||
|select age from employee where id = 1;||
|// age = 24||
||begin transaction;|
||update employee set age = 25 where id = 1;|
||// age = 25|
||commit;|
|select age from employee where id = 1;||
|// age = 25||
|commit;||

### 1.3 Phantom read
When during a transaction, you perform two queries, and the number of rows you got each time is different due to some other transaction inserting or deleting new data.
|**Transaction A**|**Transaction B**|
|:----------------|----------------:|
|begin transaction;||
|select * from employee where age > 18 and age < 24;||
|// 4 rows||
||begin transaction;|
||insert into employee(id, age) values(20);|
||commit;|
|select * from employee where age > 18 and age < 24;||
|// 5 rows||
|commit;||
### 1.4 Serialization (write skew) anomaly
[https://dba.stackexchange.com/a/315353](https://dba.stackexchange.com/a/315353)

Assume there are two rows in the table `doctors` as following:

| name | on_call |
|------|---------|
| Bob  | false   |
| Alice| false   |

There are two doctors in a hospital. The business require that there must be exactly one on call doctor every night. Both Alice and Bob open the web and see that there is no on call doctor, they therefore pick himself/herself as candidate if the other doesn't. The expected result is either Alice or Bob will be on call, not both. But for the write skew anomaly, it's possible that both will be on call.

The transactions will happen like following:  

| Transaction A (Alice) | Transaction B (Bob) |
|-----------------------|---------------------|
| begin transaction;    | begin Transaction;  |
| select name, on_call from doctors; | select name, on_call from doctors; |
| // Bob = false, Alice = false | // Bob = false, Alice = false |
| update doctors set on_call = true where name = 'Alice' and not exists (select on_call from doctors where name = 'Bob' and on_call = true); | update doctors set on_call = true where name = 'Bob' and not exists (select on_call from doctors where name = 'Alice' and on_call = true); |
| commit; | commit; |
| // Bob = true, Alice = true | // Bob = true, Alice = true |

## 2. Isolation level
### 2.1 Read uncommitted
Transaction A could see uncommitted changes from Transaction B, in other words, it allows** dirty read** to happen.
### 2.2 Read committed
Transaction A could see committed changes from Transaction B. No **dirty read** at this level, but **non-repeatable read** and **phantom read** are possible.
### 2.3 Repeatable reads
It inherits from **read-committed**, and no **non-repeatable read** in this level, it means that no matter how many times you query for a row in a single transaction, you are a warranty that all the values in the row remain unchanged.

But phantom read could happen at this level.
### 2.4 Serializable
Not any read phenomena could happen at this level, this is the highest level of isolation.

In the Serializable Isolation Level, all transactions have to execute in sequential order, it cannot be executed in parallel like in the Repeatable level.

## 3. Default Isolation level in Postgres
The default isolation level of Postgres is **Read Committed**.

There is no way to **read uncommitted** in Postgres.

**Phantom-read** is prevented even in Repeatable reads Isolation Level.
## 4. References
https://en.wikipedia.org/wiki/Isolation_(database_systems)
https://dev.to/techschoolguru/understand-isolation-levels-read-phenomena-in-mysql-postgres-c2e
https://www.postgresql.org/docs/current/transaction-iso.html
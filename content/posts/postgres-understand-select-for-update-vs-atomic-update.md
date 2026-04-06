---
date: '2026-04-06T22:31:21+07:00'
draft: false
title: 'Postgres Locking Internals: Comparing `UPDATE` and `SELECT ... FOR UPDATE`'
---
Recently I learned that Postgres holds a lock on the rows it updates during a transaction and only releases it when the transaction finishes (commit/rollback).

This surprised me because I always use `select ... for update` to lock the row before updating it in a transaction. I thought that other transactions could concurrently update the row that my current transaction is updating, causing a race condition.

For example, I have this simple `users` table:
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    balance INT
);

INSERT INTO users (id, balance)
VALUES 
    (100, 40),
    (200, 50);
```

I have two transactions running concurrently to update the same user's balance: the first transaction adds 10, and the second deducts 20.

| Time | Tx 1 | Tx 2 | Description |
| ---- | ---- | ---- | ----------- |
| 1 | | `begin;` | |
| 2 | `begin;` | | |
| 3 | `update users set balance = balance + 10 where id = 100;` <br> --- <br> *balance = 50* | | Tx 1 updates the balance of user 100, and it also acquires a lock on that row. |
| 4 | | `update users set balance = balance - 20 where id = 100;` <br> --- <br> *Tx 2 got blocked here* | Tx 2 tries to update the same row as Tx 1, but it has to wait for the lock on user 100. |
| 5 | `commit;` | *just idle* | The lock on user 100 held by Tx 1 is released. |
| 6 | | --- <br> *resumes and finishes the update, user's balance is now 30* | 
| 7 | | `commit;` |

Before learning this behavior, when I needed to update something, I always used `select ... for update` to prevent race conditions like this:

| Time | Tx 1 | Tx 2 |
| ---- | ---- | ---- |
| 1 | | `begin;` |
| 2 | `begin;` | |
| 3 | `select balance from users where id = 100 for update;` <br> --- <br> *balance = 40, the row with id 100 is explicitly locked by Tx 1* | |
| 4 | | `select balance from users where id = 100 for update;` <br> --- <br> *Tx 2 got blocked here* |
| 5 | `update users set balance = balance + 10 where id = 100;` <br> --- <br> *balance = 50* | *just idle* |
| 6 | `commit;` | *just idle* |
| 7 | | --- <br> *the select resumes, returns balance = 50* |
| 8 | | `update users set balance = balance - 20 where id = 100;` <br> --- <br> *balance = 30* |
| 9 | | `commit;` |

With that in mind, I now know that the `UPDATE` query is similar to `SELECT ... FOR UPDATE` in that they both acquire an exclusive lock on the affected row. So, do I need `select ... for update` anymore?

> Note that `UPDATE` often takes `FOR NO KEY UPDATE` row lock semantics (unless key columns are changed), while `SELECT ... FOR UPDATE` uses `FOR UPDATE` semantics.

In that case, if before the update we don't need to fetch the row to the application, we can use a single `update` query, and then we don't need `select ... for update`.

In the `Read Committed` isolation level, `select ... for update` helps our application see a consistent value of the locked rows during the transaction. For example:

| Time | Tx 1 | Tx 2 | Description |
| ---- | ---- | ---- | ----------- |
| 1 | `begin;` | | |
| 2 | | `begin;` | |
| 3 | `select balance from users where id = 100` <br> --- <br> *balance = 40* | | Initially, Tx 1 sees balance = 40 |
| 4 | | `update users set balance = 60 where id = 100;` <br> --- <br> *balance = 60* |
| 5 | | `commit;` | Tx 2 changes balance to 60 and commits |
| 6 | `select balance from users where id = 100` <br> --- <br> *balance = 60* | | Now Tx 1 sees a different value of balance |

At a broader level, to have a consistent snapshot during the transaction, we can use `Repeatable Read` isolation level. But if we are in `Read Committed` and still want a consistent view on some rows, `select ... for update` will help.

## Digging down to pg_locks

To verify this in practice, I dug down to check whether what I thought was right. Postgres has a view named `pg_locks` that contains all locks being held and waited on.

Using this command, we can see which locks a process is holding or waiting for:
```sql
SELECT 
    pid, locktype, relation::regclass AS table_name, page, tuple, transactionid, virtualxid, virtualtransaction, mode, granted, fastpath, waitstart
FROM pg_locks
WHERE pid = ?;
```
To know the pid of the process that currently handles our transaction, use the query:

```sql
select pg_backend_pid();
```
For example:
```bash
test=# select pg_backend_pid();
 pg_backend_pid 
----------------
           9038
(1 row)
```
> `pid` is the id assigned by the operating system for the process. Later in the article, there is a `backend id` which is an internal id assigned by Postgres itself, typically starting from 1.

With that setup, let's begin.

### 1. `begin;`
After using `begin;` to begin a transaction, open a different terminal and query for its locks:

```bash
  pid  |  locktype  | table_name | page | tuple | transactionid | virtualxid | virtualtransaction |     mode      | granted | fastpath | waitstart
-------+------------+------------+------+-------+---------------+------------+--------------------+---------------+---------+----------+-----------
 13054 | virtualxid |            |      |       |               | 7/53394    | 7/53394            | ExclusiveLock | t       | t        |
(1 row)
```
The transaction holds an `ExclusiveLock` on itself, on its virtualxid.
A virtual xid is made of:
- Backend ID (not PID)
- Local sequencer number

The virtual xid `7/53394` means backend id `7` and sequencer number `53394`. The next virtual xid on this process is probably `7/53395`.

The lock on virtual xid at the beginning of the transaction doesn't mean it protects any data or structure. It just registers the lock, so that later on, other transactions might wait on this lock for it to finish.

### 2. `select vs update`
After looking at `begin;`, let's compare `select` and `update`.
#### 2.1 `select balance from users where id = ?`
Let's query the balance of a user, without using `select ... for update`. Here is the lock view:

```bash
  pid  |  locktype  | table_name | page | tuple | transactionid | virtualxid | virtualtransaction |      mode       | granted | fastpath | waitstart
-------+------------+------------+------+-------+---------------+------------+--------------------+-----------------+---------+----------+-----------
 13054 | relation   | users_pkey |      |       |               |            | 7/53394            | AccessShareLock | t       | t        |
 13054 | relation   | users      |      |       |               |            | 7/53394            | AccessShareLock | t       | t        |
 13054 | virtualxid |            |      |       |               | 7/53394    | 7/53394            | ExclusiveLock   | t       | t        |
(3 rows)
```
It acquired two more `AccessShareLock` locks on the table `users` and the primary key index. This is to block DDL operations that require an `AccessExclusiveLock`, such as `DROP TABLE` or `REINDEX`.
#### 2.2 `select balance from users where id = ? for update`
```bash
  pid  |   locktype    | table_name | page | tuple | transactionid | virtualxid | virtualtransaction |      mode       | granted | fastpath | waitstart
-------+---------------+------------+------+-------+---------------+------------+--------------------+-----------------+---------+----------+-----------
 13054 | relation      | users_pkey |      |       |               |            | 7/53394            | AccessShareLock | t       | t        |
 13054 | relation      | users_pkey |      |       |               |            | 7/53394            | RowShareLock    | t       | t        |
 13054 | relation      | users      |      |       |               |            | 7/53394            | AccessShareLock | t       | t        |
 13054 | relation      | users      |      |       |               |            | 7/53394            | RowShareLock    | t       | t        |
 13054 | virtualxid    |            |      |       |               | 7/53394    | 7/53394            | ExclusiveLock   | t       | t        |
 13054 | transactionid |            |      |       |        316069 |            | 7/53394            | ExclusiveLock   | t       | f        |
(6 rows)
```
It acquired an additional `RowShareLock` on the table and index of `users`. (Note that `AccessShareLock` was acquired previously on a normal select query.)

It acquired an `ExclusiveLock` on its `transactionid`; this lock is on itself, just like the `virtualxid`.

In Postgres, only when the transaction modifies the heap page does it get assigned a real `transactionid`. To modify the heap page, it can do one of these: insert, `update`, `delete`, `select ... for update`, `select ... for share`, ...

While `virtualxid` is valid internally in each process, `transactionid` is a `32 bit integer` valid globally, and it is monotonic (but not monotonic increasing forever, only until wraparound point).

The lock on its transaction id `316069` informs other transactions that all rows edited by `316069` (heap tuples) are locked; if they want to edit them, they must wait.

**Bonus:** *How does the lock view look when another transaction tries the same row?*

Start a second transaction and run `update` on the same row as the first.
```bash
  pid  |   locktype    | table_name | page | tuple | transactionid | virtualxid | virtualtransaction |       mode       | granted | fastpath |           waitstart
-------+---------------+------------+------+-------+---------------+------------+--------------------+------------------+---------+----------+-------------------------------
 30532 | relation      | users_pkey |      |       |               |            | 5/34951            | RowExclusiveLock | t       | t        |
 30532 | relation      | users      |      |       |               |            | 5/34951            | RowExclusiveLock | t       | t        |
 30532 | virtualxid    |            |      |       |               | 5/34951    | 5/34951            | ExclusiveLock    | t       | t        |
 30532 | transactionid |            |      |       |        316069 |            | 5/34951            | ShareLock        | f       | f        | 2026-04-06 14:07:13.531661+00
 30532 | transactionid |            |      |       |        316070 |            | 5/34951            | ExclusiveLock    | t       | f        |
 30532 | tuple         | users      |    0 |     6 |               |            | 5/34951            | ExclusiveLock    | t       | f        |
(6 rows)
```

- It's waiting for the lock on transaction id `316069`, `granted` is false and `waitstart` is `2026-04-06 14:07:13.531661+00`.
- Notice that it successfully acquired a lock on a heap tuple at slot (0,6) of table `users`.

**Bonus:** *What happens if a third transaction is waiting on the same row?*

Start a third transaction and do the same thing as the second.


```bash
  pid  |   locktype    | table_name | page | tuple | transactionid | virtualxid | virtualtransaction |       mode       | granted | fastpath |           waitstart
-------+---------------+------------+------+-------+---------------+------------+--------------------+------------------+---------+----------+-------------------------------
 29686 | relation      | users_pkey |      |       |               |            | 6/34773            | RowExclusiveLock | t       | t        |
 29686 | relation      | users      |      |       |               |            | 6/34773            | RowExclusiveLock | t       | t        |
 29686 | virtualxid    |            |      |       |               | 6/34773    | 6/34773            | ExclusiveLock    | t       | t        |
 29686 | tuple         | users      |    0 |     6 |               |            | 6/34773            | ExclusiveLock    | f       | f        | 2026-04-06 14:10:57.326324+00
 29686 | transactionid |            |      |       |        316071 |            | 6/34773            | ExclusiveLock    | t       | f        |
(5 rows)
```
- The third transaction is now waiting for the lock on the tuple at slot (0,6) of table `users`. If the first transaction finishes with the tuple (0,6), the second transaction typically takes its turn before the third.
- *Bonus:* many waits on `transactionid` and `tuple` are a signal of high locking contention.

#### 2.3 `update users set balance = balance + 10 where id = ?`

```bash
  pid  |   locktype    | table_name | page | tuple | transactionid | virtualxid | virtualtransaction |       mode       | granted | fastpath | waitstart
-------+---------------+------------+------+-------+---------------+------------+--------------------+------------------+---------+----------+-----------
 13054 | relation      | users_pkey |      |       |               |            | 7/53394            | AccessShareLock  | t       | t        |
 13054 | relation      | users_pkey |      |       |               |            | 7/53394            | RowShareLock     | t       | t        |
 13054 | relation      | users_pkey |      |       |               |            | 7/53394            | RowExclusiveLock | t       | t        |
 13054 | relation      | users      |      |       |               |            | 7/53394            | AccessShareLock  | t       | t        |
 13054 | relation      | users      |      |       |               |            | 7/53394            | RowShareLock     | t       | t        |
 13054 | relation      | users      |      |       |               |            | 7/53394            | RowExclusiveLock | t       | t        |
 13054 | virtualxid    |            |      |       |               | 7/53394    | 7/53394            | ExclusiveLock    | t       | t        |
 13054 | transactionid |            |      |       |        316069 |            | 7/53394            | ExclusiveLock    | t       | f        |
(8 rows)
```
The locks it acquired are almost identical to the `select ... for update`. Except that this time it acquired additional `RowExclusiveLock` instead of `RowShareLock` on the table and index of `users`.
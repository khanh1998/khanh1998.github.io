---
date: '2025-12-15T22:15:57+07:00'
draft: true
title: 'Postgres MVCC - Multi Version Concurrency Control'
mermaid: true
---

## TL;DR
- MVCC gives each transaction a consistent view of data without copying the whole database
- It keeps multiple tuple versions of the same logical row and use snapshot to decide visibility

## Prerequisites
This article assumes the reader aware about isolation levels in database.

**References**
- https://www.postgresql.org/docs/current/transaction-iso.html
- https://khanh1998.github.io/posts/isolation-level-and-read-phenomena/

## 1) Repeatable Read Example

There is a row containing user's balance on table `user_balance` as bellow:
| id | user_id | balance |
| -- | ------- | ------- |
| 1  | 1001    | 5000    |

Assume there are two transactions working on the that same row concurrently:

| Time | TX 1 | TX 2 |
| -----| ---- | ---- |
| 1 | `BEGIN ISOLATION LEVEL REPEATABLE READ;`| `BEGIN;` <br> // read committed by default|
| 2 | `select balance from user_balance where user_id = 1001;` <br> // see 5000 as user's balance ||
| 3 || `update user_balance set balance = balance + 1000 where user_id = 1001;`  |
| 4 || `COMMIT;` <br> // user's balance now is 6000|
| 5 | `select balance from user_balance where user_id = 1001;` <br> // as expected, still see user's balance 5000  ||

From the above example, TX 1 running on repeatable read isolation level. At first, it see user's balance 5000. After TX 2 committed a new user's balance, it still see user's balance 5000. No matter what other transactions do, TX 1 always see user's balance 5000 (unless it change user's balance by it self). That is what snapshot mean, helping transaction can see a consistent view of data.

In Postgres, serializable is also behave the same way, the transaction also see a consistent snapshot.

How do they implement the consistent snapshot for each transaction? What is a snapshot? Did it make a copy of the whole database at the time the transaction begin, so it can refer to the old values, or they have a better way, let's dive in.


## Postgres Internal
### Transaction ID (aka XID)
Postgres uses 32 bits integer as xid. There is a global counter for xid, it is monotonically increasing, therefore former tx will have smaller later transaction (Well, it's actually [more complex](https://www.postgresql.org/docs/18/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND) than that, but for the sake of this article, let's just think about it that way)


Postgres assign xid to transaction lazily, it means if you `BEGIN` a transaction, Postgres won't assign an xid to it until the first create, update or delete query. That also mean, if you `BEGIN` a transaction that read only, then it won't get an transaction xid. Because xid only necessary for create, update and delete, as it need to record the xid of the transaction which do the action.

A single create, update or delete - without `BEGIN` - also got an xid.

To know if the current transaction has assigned an xid or not, use the function `pg_current_xact_id_if_assigned()`.

The function `pg_current_xact_id()` is a bit different, it will force Postgres to assign to the current transaction an xid regardless if it need it or not.

**References**
- https://www.postgresql.org/docs/current/transaction-id.html
- https://pgpedia.info/p/pg_current_xact_id.html
- https://pgpedia.info/p/pg_current_xact_id_if_assigned.html
### Inspecting the Heap Page
When creating an table, we define the columns, each row in that table will have all of those defined columns. But internally, each row has a few more columns used by the system itself.

`ctid`, `xmin` and `xmax` are three of those hidden system columns that we need in this article.

Now let's create the table `user_balance`. 

> **Note**: I use the tool `psql` in this post.
> We can start a new Postgres container on Docker, open the terminal, and use `psql -U postgres -d postgres` to interact with the database.
> The first `postgres` is username, the second `postgres` is the database name.

```sql
CREATE TABLE user_balance (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    balance NUMERIC(12, 2) NOT NULL
);
```
Begin a new transaction and insert a new row to the table. Notice that the id of transaction is 767.
```sh
postgres=# begin;
BEGIN
postgres=*# SELECT pg_current_xact_id();
 pg_current_xact_id 
--------------------
                767
(1 row)

postgres=*# INSERT INTO user_balance (user_id, balance)
VALUES (1001, 5000);
INSERT 0 1
postgres=*# commit;
COMMIT
```

Now we can query the columns that we defined along with the system columns:
```sql
postgres=# select ctid, xmin, xmax, user_id, balance from user_balance where user_id = 1001;
 ctid  | xmin | xmax | user_id | balance 
-------+------+------+---------+---------
 (0,1) |  767 |    0 |    1001 | 5000.00
(1 row)
```
- `ctid` aka current tuple ID, physical location of the row is at offset 0 of the heap page 0 
- `xmin` is id of the transaction inserting the row version, 767 matches the transaction id above. 
- `xmax` is the id of the transaction delete the row version, the value of `0` means the row version isn't deleted by any transaction yet, it's the latest version of the row.

Now, let's try to update the row.
```sh
postgres=# update user_balance set balance = balance + 1000 where user_id = 1001;
UPDATE 1
postgres=# select ctid, xmin, xmax, user_id, balance from user_balance where user_id = 1001;
 ctid  | xmin | xmax | user_id | balance 
-------+------+------+---------+---------
 (0,2) |  768 |    0 |    1001 | 6000.00
(1 row)
```
- `ctid` current physical location of the row now is moved to offset 2 of the heap page 0. Notice that the previous version is in offset 1.
- `xmin` is changed to 768 which is the id of the transaction updating the row.

**Important**: The data rows we can get by a select query like above are usually referred as logical row. Logical row is backed by tuples (physical rows). In other words, logical rows are what you see, tuples (physical rows) are what stored by the database. It is like you see number 2, but the database really store 1+1.

Tuples are stored in a space named heap, heap are divided into pages, 8KB each. We can use `pageinspect` tool to see what are really stored in the heap page. 
```sh
postgres=# CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE EXTENSION
postgres=# SELECT *
FROM heap_page_items(get_raw_page('user_balance', 0));
```
This query will show all data in the heap page `0` of the table `user_balance`

The select will show a result of many columns like this:
```
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |                    t_data                    
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------------------------------------
  1 |   8144 |        1 |     45 |    767 |    768 |        0 | (0,2)  |       16387 |       1282 |     24 |        |       | \x0100000000000000e9030000000000000b00818813
  2 |   8096 |        1 |     45 |    768 |      0 |        0 | (0,2)  |       32771 |      10498 |     24 |        |       | \x0100000000000000e9030000000000000b00817017
(2 rows)
```

There are many columns, now let's focus on some important ones related to this post.
```sql
postgres=# SELECT lp, t_xmin, t_xmax, t_data
FROM heap_page_items(get_raw_page('user_balance', 0));
 lp | t_xmin | t_xmax |                    t_data                    
----+--------+--------+----------------------------------------------
  1 |    767 |    768 | \x0100000000000000e9030000000000000b00818813
  2 |    768 |      0 | \x0100000000000000e9030000000000000b00817017
(2 rows)
```
From the normal query, we see one (logical) row, but under the hood, there are two tuples (physical) of that row.

- `lp` is line pointer, or the offset in the heap page. the first version (balance = 5000) is in offset 1, the second version (balance = 6000) is on offset 2.
- `t_data` containing encoded data of row version. The data of the row like `id`, `user_id`, `balance` are encoded and stored there.
- `t_xmin`and `t_xmax` are similar to `xmin` and `xmax` we seen above.
- `t_ctid` 

Transaction 767, created the row, inserting tuple at offset 1.
Transaction 768, updated the row, by updating the t_max to 768, and inserting a new tuple for new value at offset 2.


So, in sum up, when we update a row in postgres, it doesn't update the row in place, it rather create a new version for it.

**References**
- https://www.postgresql.org/docs/current/pageinspect.html#PAGEINSPECT-HEAP-FUNCS

### Snapshot
Is snapshot a copy all all data? nah, it's defined by a few numbers.

```bash
postgres=*# SELECT pg_current_snapshot();
 pg_current_snapshot 
---------------------
 778:782:778,779
(1 row)
```
The output is in the format: `xmin:xmax:xip_list`:
- `xmin`: lowest active transaction id.
- `xmax`: the next transaction id to be assigned
- `xip_list`: list of active transaction id at the snapshot time between xmin and xmax (xmin <= id < xmax)

Let's consider the snapshot `778:782:778,779`. At the time the snapshot is created, smallest xid of active transactions is 778, 782 is the next xid to be assigned to new transaction, between 778 and 785 [778,782), there are two active transactions 778 and 779.

{{< mermaid fontSize="24px" >}}
timeline
    title Transaction XID Timeline
    section Before Snapshot
        776 : Rollback
        777 : Committed
    section The Snapshot
        778 : Action
            : xmin
        779 : active
        780 : Committed
        781 : Committed
        782 : Not Assigned Yet: Xmax
    section After Snapshot
        783 : Not assigned yet
{{< /mermaid >}}

The way snapshot is created:
- Read committed: get new snapshot at start of any query in the transaction. Each query in the transaction may see a different snapshot.
- Repeatable Read, Serializable: Get a snapshot at start of the first query, and reuse that snapshot through out the transaction. All queries in the same transaction see the same snapshot.

### MVCC rule

Now let's put it together, each heap tuple has `xmin` and `xmax` storing xid of the transaction creating, updating or deleting that tuple. Each transaction will have a snapshot. Based on the snapshot, xmin and xmax the database can decide which tuples are visible to the transaction.

To decide if a tuple in heap page are visible to the transaction or not: the general logic are:

**Phase 1**: Checking tuple's `xmin`
if tuple.xmin = current transaction xid
if tuple.xmin < snapshot.xmin → visible
if tuple.xmax > snapshot.xmax → invisible
if tuple.xmax is in snapshot.xip_list → invisible

**Phase 2**: Checking tuple's `xmax`

xmin is 778
xmax is 785
xip_list is 778, 779

{{< mermaid fontSize="24px" >}}
timeline
    title Transaction XID
    section Before Snapshot
        777 : Committed
    section The Snapshot
        778 : Xmin
            : active
        779 : active
        780 : Committed
        781 : Committed
        782 : Committed
        783 : Xmax : Not assigned yet
    section After Snapshot
        784 : Not assigned yet
{{< /mermaid >}}


**Important**: In this article, I can only describe a roughly idea of MVCC. It is in fact much more complicated than that. For example, it need to handle its own changes, rollback tuples. For full detail, please check out the code [HeapTupleSatisfiesMVCC](https://github.com/postgres/postgres/blob/aa082bed0b6433b58815683dde425bce57ed931c/src/backend/access/heap/heapam_visibility.c#L867-L869), it will give you the full detail of MVCC rules.

**References**
- `HeapTupleSatisfiesMVCC` code: https://github.com/postgres/postgres/blob/aa082bed0b6433b58815683dde425bce57ed931c/src/backend/access/heap/heapam_visibility.c#L867-L869
- https://www.postgresql.org/docs/current/functions-info.html#FUNCTIONS-INFO-SNAPSHOT
- https://www.interdb.jp/pg/pgsql05/05.html
- https://mbukowicz.github.io/databases/2020/05/01/snapshot-isolation-in-postgresql.html
- https://www.postgresql.org/files/developer/concurrency.pdf

### Example

| TX 0 | TX 1 | TX 2 | TX 3 |
| ---- | ---- | ---- | ---- |
| `BEGIN;` ||||
| `SELECT pg_current_xact_id();` <br> *771* ||||
| `SELECT pg_current_snapshot();` <br> *771* ||||
| `select balance from user_balance where user_id = 1001;` <br> // 6000 ||||
| `update user_balance set balance = balance + 1000 where user_id = 1001;` <br> 7000||||
| | `BEGIN ISOLATION LEVEL REPEATABLE READ;` |||
| || `BEGIN;` ||
| || `SELECT pg_current_xact_id();` <br> 770 ||
| || `SELECT pg_current_snapshot();` <br> 770 ||
| || `select balance from user_balance where user_id = 1001` <br> 6000 ||
| || `update user_balance set balance = balance + 1000 where user_id = 1001;` <br> 7000||
| || `COMMIT;` ||`
| ||| `BEGIN;`|
| ||| `SELECT pg_current_xact_id();` <br> 771|
| | `select balance from user_balance where user_id = 1001` <br> 6000 |||
| | `SELECT pg_current_xact_id();` <br> *772* |||
| | `select balance from user_balance where user_id = 1001` <br> 6000 |||
| ||| `select balance from user_balance where user_id = 1001` <br> 6000 |
| ||| `select balance from user_balance where user_id = 1001` <br> 7000 |
| ||| `update user_balance set balance = balance + 1000 where user_id = 1001;` <br> 8000|
| ||| `COMMIT;` |
| | `select balance from user_balance where user_id = 1001` <br> 6000 |||
| | `COMMIT;` |||
| `COMMIT;` |||

Let's continue with our example. This time create three transactions 1, 2 and 3.
The transaction 1 running on repeatable read isolation level, so it expects to see a consistent view of user balance. At the begin, it see user's balance as 6000, through out the transaction, it must see 6000.
Along the transaction 1, transaction 2 and 3 will modify the value of the user's balance, but the changes won't affect to the view of transaction 1.

When the transaction 1 begin, the page layout look like this:
```sql
postgres=# SELECT lp, t_xmin, t_xmax, t_data
FROM heap_page_items(get_raw_page('user_balance', 0));
 lp | t_xmin | t_xmax |                    t_data                    
----+--------+--------+----------------------------------------------
  1 |    767 |    768 | \x0100000000000000e9030000000000000b00818813
  2 |    768 |      0 | \x0100000000000000e9030000000000000b00817017
(2 rows)
```
Since transaction id of the transaction 1 is 769, so it will see the latest version of the row.

Then the transaction 2 come along and edit the user balance, the heap page layout now look like:
```sql
postgres=# SELECT lp, t_xmin, t_xmax, t_data
FROM heap_page_items(get_raw_page('user_balance', 0));
 lp | t_xmin | t_xmax |                    t_data
----+--------+--------+----------------------------------------------
  1 |    767 |    768 | \x0100000000000000e9030000000000000b00818813
  2 |    768 |    776 | \x0100000000000000e9030000000000000b00817017
  3 |    776 |      0 | \x0100000000000000e9030000000000000b0081581b
(3 rows)
```

and the transaction 3 come:

```sql
postgres=# SELECT lp, t_xmin, t_xmax, t_data
FROM heap_page_items(get_raw_page('user_balance', 0));
 lp | t_xmin | t_xmax |                    t_data
----+--------+--------+----------------------------------------------
  1 |    767 |    768 | \x0100000000000000e9030000000000000b00818813
  2 |    768 |    776 | \x0100000000000000e9030000000000000b00817017
  3 |    776 |    777 | \x0100000000000000e9030000000000000b0081581b
  4 |    777 |      0 | \x0100000000000000e9030000000000000b0081401f
(4 rows)
```

# auto vacumn

```sql
postgres=# vacuum user_balance;
VACUUM
postgres=# SELECT lp, t_xmin, t_xmax, t_data
FROM heap_page_items(get_raw_page('user_balance', 0));
 lp | t_xmin | t_xmax |                    t_data                    
----+--------+--------+----------------------------------------------
  1 |        |        | 
  2 |        |        | 
  3 |        |        | 
  4 |        |        | 
  5 |        |        | 
  6 |    777 |      0 | \x0100000000000000e9030000000000000b0081401f
(6 rows)

postgres=# select ctid, xmin, xmax, user_id, balance from user_balance where user_id = 1001;
 ctid  | xmin | xmax | user_id | balance 
-------+------+------+---------+---------
 (0,6) |  777 |    0 |    1001 | 8000.00
(1 row)
```

# table bloat

# long running transaction
how do long running transaction block vacuum

assume tx 1 has snapshot 100:110:100,101,102

that mean it can't see tuples from tx 110 and above
but it can see tuples from tx 103 to 109, so the vacuum can't delete these tuple

to make thing simple, postgres just take the xmin from all registered snapshot as `oldestXmin`, any tuples above this points can't be vacuum


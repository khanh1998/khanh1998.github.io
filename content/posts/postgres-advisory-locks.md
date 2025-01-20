---
date: '2025-01-20T11:56:55+07:00'
draft: false
title: 'Postgres Advisory Locks'
tags: ['db', 'postgres-for-everything']
---

This post is part of the series *Postgres for Everything*.

Nowadays, we typically develop stateless applications, making it easier to scale them horizontally. However, locking is an essential tool to prevent race conditions in software development. In applications with multiple instances, programming language-level locks are insufficient because they only work locally within an instance. A centralized locking mechanism, valid across all instances, is required.

## Redis

### SETNX
You can use [`SETNX`](https://redis.io/docs/latest/commands/setnx/) to implement a simple lock. `SETNX` sets a value for a key only if the key does not already exist.

The key acts as the lock. Initially, it does not exist, so the fastest client can acquire it by setting a value. Other clients must wait until the key no longer exists to acquire the lock. Once the job is complete, the client releases the lock by deleting the key, giving other clients a chance to acquire it.

```plaintext
SETNX key value
```

`SETNX` does not allow setting an expiration time atomically when the key is created. Adding an expiration time is crucial to mitigate deadlocks and prevent crashed clients from holding locks indefinitely. You can achieve this using [`SET`](https://redis.io/docs/latest/commands/set/):

```plaintext
SET key value NX EX 10
```

This approach works well when your Redis deployment consists of a single master node. However, issues arise in a master-replica setup. If the master crashes before a key is replicated to its replicas, the promoted replica might allow a new client to `SETNX` on the same key, resulting in two clients holding the same lock.

### Redlock

To overcome the limitations of `SETNX`, Redis offers [Redlock](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/). Redlock works on multi-master clusters, improving the availability of the locking mechanism.

In a cluster with N nodes, Redlock requires locking N/2 + 1 nodes, tolerating up to N/2 - 1 node failures. For example, in a 5-node cluster, Redlock remains functional even if 2 nodes fail.

However, Redlock is not guaranteed to be safe in all scenarios, as discussed in [Martin Kleppmann’s blog](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html). Therefore, Redlock might not be suitable for critical systems like banking or financial applications.

## Postgres

Redis locks are convenient but may lack the safety required for critical tasks. If you already use Postgres in your project, it can be a great choice for centralized locking.

```sql
CREATE TABLE user_balance (
    name TEXT PRIMARY KEY,       -- Name of the user, serves as the primary key
    balance NUMERIC(15, 2) NOT NULL DEFAULT 0.00 -- Balance with two decimal places
);
INSERT INTO user_balance (name, balance) VALUES ('Alice', 100), ('Bob', 200);
```

### Row-Level Locks
Using the `SELECT ... FOR UPDATE` syntax, you can acquire an exclusive lock on rows. These locks are automatically released at the end of the transaction. This mechanism is often combined with transactions for atomic updates.

For more details, see [PostgreSQL Row-Level Locks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS).

#### Demonstration
| Time | Transaction 1 | Transaction 2 |
|------|---------------|---------------|
| 1.0  |               | `BEGIN;`      |
| 2.0  | `BEGIN;`      |               |
| 3.0  | `SELECT balance FROM user_balance WHERE name = 'Alice' FOR UPDATE;` | |
| 3.1  | Alice’s balance = 100 |        |
| 4.0  |               | `SELECT balance FROM user_balance WHERE name = 'Alice' FOR UPDATE;` |
| 4.1  |               | Blocked (Alice’s row is locked by TX1) |
| 5.0  | `SELECT balance FROM user_balance WHERE name = 'Bob' FOR UPDATE;` | |
| 5.1  | Bob’s balance = 200 |        |
| 6.0  | `UPDATE user_balance SET balance = balance - 100 WHERE name = 'Alice';` | |
| 7.0  | `UPDATE user_balance SET balance = balance + 100 WHERE name = 'Bob';` | |
| 8.0  | `COMMIT;`      |               |
| 9.0  |               | Alice’s balance = 0 |
| 10.0 |               | Continue      |

#### Deadlock Example
Deadlocks can occur when transactions acquire locks in an inconsistent order. For instance:

| Time | Transaction 1 | Transaction 2 |
|------|---------------|---------------|
| 1.0  |               | `BEGIN;`      |
| 2.0  | `BEGIN;`      |               |
| 3.0  | `SELECT balance FROM user_balance WHERE name = 'Alice' FOR UPDATE;` | |
| 4.0  |               | `SELECT balance FROM user_balance WHERE name = 'Bob' FOR UPDATE;` |
| 5.0  | `SELECT balance FROM user_balance WHERE name = 'Bob' FOR UPDATE;` | Blocked |
| 6.0  |               | `SELECT balance FROM user_balance WHERE name = 'Alice' FOR UPDATE;` |
| 7.0  | Deadlock       | Deadlock      |

To avoid deadlocks, acquire locks in a deterministic order (e.g., by ascending username).

#### Optimistic Locking
Optimistic locking assumes no concurrent modifications. It checks data integrity at the end of the transaction, rejecting changes if data has been altered. This approach is implemented using a `version` column, incremented on updates.

##### Successful Scenario
| Time | Transaction 1 | Transaction 2 |
|------|---------------|---------------|
| 1.0  |               | `BEGIN;`      |
| 2.0  | `BEGIN;`      |               |
| 3.0  | `SELECT balance, version FROM user_balance WHERE name = 'Alice';` | |
| 5.0  | `UPDATE user_balance SET balance = balance - 100, version = version + 1 WHERE name = 'Alice' AND version = 0;` | |
| 7.0  | `COMMIT;`      |               |

##### Failure Scenario
| Time | Transaction 1 | Transaction 2 |
|------|---------------|---------------|
| 1.0  |               | `BEGIN;`      |
| 5.0  | `UPDATE user_balance SET balance = balance - 100, version = version + 1 WHERE name = 'Alice' AND version = 0;` | |
| 6.0  |               | `UPDATE user_balance SET balance = 100, version = version + 1 WHERE name = 'Bob';` |
| 7.0  | Update failed (version mismatch) | |
| 8.0  | `ROLLBACK;`    |               |

Optimistic locking can also be implemented without a `version` column by validating data directly in the `UPDATE` query:

```sql
UPDATE user_balance SET name = 'Alice Watson' WHERE id = 123 AND name = 'Alice';
```
## Advisory Locks
Row-level locks in PostgreSQL are useful, but what if the resource you need to lock doesn’t correspond to a database row? PostgreSQL provides an alternative: [advisory locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS). These locks are application-defined and can operate independently of the database schema. Advisory locks can be acquired at two levels: transaction and session.

### Transaction-Level Locks
**Exclusive Locks:**  
You can acquire an exclusive lock at the transaction level using the `pg_advisory_xact_lock` function. This is a blocking function, meaning that if another transaction already holds the lock, the current transaction will wait until the lock becomes available:

```sql
BEGIN;
SELECT pg_advisory_xact_lock(123);
COMMIT;
```

For a non-blocking version, use `pg_try_advisory_xact_lock`. It returns `TRUE` if the lock is acquired and `FALSE` otherwise.

**Shared Locks:**  
The `pg_advisory_xact_lock` function provides exclusive locks, allowing only one transaction to hold the lock at a time. For shared access, use `pg_advisory_xact_lock_shared`. This allows multiple transactions to hold the lock simultaneously, making it suitable for read-only tasks. However, while shared locks are held, no exclusive lock can be acquired on the same key.

| Time | Transaction 1                              | Transaction 2                              |
|------|-------------------------------------------|-------------------------------------------|
| 1    | `BEGIN;`                                   |                                           |
| 2    |                                           | `BEGIN;`                                   |
| 3    | Acquire shared lock: `select pg_advisory_xact_lock_shared(123);` |                                           |
| 4    |                                           | Try to acquire exclusive lock: `select pg_advisory_xact_lock(123);` |
|4.1|| Get blocked|
| 5    | `commit;`: shared lock released              | Exclusive lock acquired after release     |

**Lock Release:**  
All transaction-level advisory locks are automatically released when the transaction ends, whether through `COMMIT` or `ROLLBACK`.

**Locking Costs:**  
Advisory locks are lightweight and impose less overhead compared to row-level locks. For example:

```sql
postgres=# explain analyze select pg_advisory_lock(123);
                                     QUERY PLAN
------------------------------------------------------------------------------------
 Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.010..0.012 rows=1 loops=1)
 Planning Time: 0.111 ms
 Execution Time: 0.046 ms
(3 rows)

postgres=# explain analyze select * from user_balance where name = 'Alice' for update;
                                                              QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------
 LockRows  (cost=0.15..8.18 rows=1 width=60) (actual time=0.201..0.207 rows=1 loops=1)
   ->  Index Scan using user_balance_pkey on user_balance  (cost=0.15..8.17 rows=1 width=60) (actual time=0.061..0.066 rows=1 loops=1)
         Index Cond: (name = 'Alice'::text)
 Planning Time: 0.218 ms
 Execution Time: 0.258 ms
(5 rows)
```

### Handling Non-Integer Keys
Advisory locks only accept integer keys. To lock on a string key, hash it into an integer first:

```sql
SELECT pg_advisory_xact_lock(hashtext('my_unique_lock'));
```

### Session-Level Locks
Session-level locks persist across multiple transactions until explicitly released or the session ends. They are less commonly used but can be explored further in the [PostgreSQL documentation](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS).

### Monitoring Locks
To view active advisory locks, query the `pg_locks` system catalog:

```sql
SELECT * FROM pg_locks WHERE locktype = 'advisory';
```

## Conclusion
If Postgres is part of your tech stack, it’s an excellent choice for centralized locking, offering robust transaction management. Redis `SETNX` is fast but limited to single-node deployments. Redis Redlock provides high availability but lacks safety for critical applications. For highly available and safe distributed locking, consider Etcd, Consul, or ZooKeeper.


---
date: '2026-04-11T12:01:56+07:00'
draft: false
title: 'Postgres: How Many Connections to Open? Benchmark'
chartjs: true
math: true
---

## The question
I've heard many people said that, in postgres, each connection is a process, so it consume more resource than thread, context switch between processes is also costlier. When there are too many processes, the context switch overhead will cause the performance go down.

Those statements are intuitive. But i want to observe the system from when number of connection small to large.

## The setup

### General idea
The idea is simple. I created two VPSs, one to run pgBench, another to run Postgres, both in the same region to reduce network latency. I created few tables on postgres, then run pgBench to apply some workloads to that tables. Each run I use a different number of connection, from small to large, during the run, I will collect metrics to later comparing.

### Table setup
```sql
CREATE TABLE users (
    id         BIGSERIAL   PRIMARY KEY,
    name       TEXT        NOT NULL,
    email      TEXT        UNIQUE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE wallets (
    id         BIGSERIAL      PRIMARY KEY,
    user_id    BIGINT         NOT NULL REFERENCES users(id),
    currency   CHAR(3)        NOT NULL DEFAULT 'USD',
    balance    NUMERIC(18, 2) NOT NULL DEFAULT 0,
    status     TEXT           NOT NULL DEFAULT 'active'
                   CHECK (status IN ('active', 'frozen', 'closed')),
    updated_at TIMESTAMPTZ    NOT NULL DEFAULT NOW(),
    created_at TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);

CREATE TABLE transactions (
    id              BIGSERIAL      PRIMARY KEY,
    idempotency_key UUID           NOT NULL UNIQUE,
    type            TEXT           NOT NULL
                        CHECK (type IN ('transfer', 'topup', 'withdrawal')),
    amount          NUMERIC(18, 2) NOT NULL CHECK (amount > 0),
    currency        CHAR(3)        NOT NULL,
    status          TEXT           NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'completed', 'failed', 'reversed')),
    description     TEXT,
    created_at      TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);

CREATE TABLE ledger_entries (
    id             BIGSERIAL      PRIMARY KEY,
    transaction_id BIGINT         NOT NULL REFERENCES transactions(id),
    wallet_id      BIGINT         NOT NULL REFERENCES wallets(id),
    amount         NUMERIC(18, 2) NOT NULL CHECK (amount > 0),
    direction      TEXT           NOT NULL CHECK (direction IN ('debit', 'credit')),
    created_at     TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);
```

### Workload
Full detail of the sql script can be found at: https://github.com/khanh1998/pg-connection-bench

#### Write heavy
```sql
\set sender_id   random(1, {{NUM_USERS}})
\set receiver_id random(1, {{NUM_USERS}})
\set amount      random(1, 50)

BEGIN;

SELECT id, balance
FROM wallets
WHERE id IN (:sender_id, :receiver_id)
ORDER BY id
FOR UPDATE;

SELECT floor(balance)::int AS sender_balance
FROM wallets WHERE id = :sender_id
\gset

\if :sender_balance < 1
ROLLBACK;
\else
UPDATE wallets SET balance = balance - :amount, updated_at = NOW() WHERE id = :sender_id;
UPDATE wallets SET balance = balance + :amount, updated_at = NOW() WHERE id = :receiver_id;

INSERT INTO transactions (idempotency_key, type, amount, currency, status)
VALUES (gen_random_uuid(), 'transfer', :amount, 'USD', 'completed')
RETURNING id \gset txn_

INSERT INTO ledger_entries (transaction_id, wallet_id, amount, direction)
VALUES
    (:txn_id, :sender_id,   :amount::numeric, 'debit'),
    (:txn_id, :receiver_id, :amount::numeric, 'credit');

COMMIT;
\endif
```

### Read-Write balance


### Collected metrics

pgbench: tps, latency, latency stddev
postgres:- pg_stat_activity, 
- pg_stat_statements
os: /proc/vmstat
kernel: `perf` {cpu-cycles,instructions,ref-cycles,bus-cycles},{uops_issued.any,uops_issued.stall_cycles,uops_retired.slots,uops_retired.stall_cycles},{cycle_activity.stalls_total,cycle_activity.stalls_mem_any,cycle_activity.stalls_l1d_miss,cycle_activity.stalls_l3_miss},{resource_stalls.sb,mem_inst_retired.lock_loads,cache-references,cache-misses},msr/aperf/,msr/mperf/,cpu-clock,task-clock,context-switches,cpu-migrations,major-faults,minor-faults

### Hardware
CherryServers
- pgBench run on VPS: 8 vCPU (Intel Xeon Processor (Icelake)), 31 GB RAM, 200 GB NVMe
- Postgres run on bare metal: 8 cores 16 threads (Intel Xeon Gold 5315Y @ 3.20GHz), 31 GB RAM, 250 GB NVMe (Raid 1)

*Postgres was running on bare metal for a stable performance*
*Both instances was on the same region to reduce network delay*

### Software
pgbench

Postgres 18 running with config from PgTune: 
```
max_connections = 500
shared_buffers = 8GB
effective_cache_size = 24GB
maintenance_work_mem = 2GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 1000
work_mem = 21466kB
huge_pages = try
jit = off
wal_compression = lz4
autovacuum_work_mem = 2GB
io_method = io_uring
min_wal_size = 2GB
max_wal_size = 8GB
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4
```

## Benchmark scenarios
The same setup will be executed against these scenarios:
16, 32, 64, 128, 256, 512 and 1024 connections.

## The observation

we gonna going from top to the bottom

### Write heavy workload
#### pgBench metrics

| Metric | c16 | c32 | c64 | c128 | c256 | c512 | c1024 |
|---|---|---|---|---|---|---|---|
| TPS | 3085.11 | 6365.24 | 9169.62 | 11808.88 | 13150.34 | 12719.56 | 11967.83 |
| Avg Latency (ms) | 5.186 | 5.019 | 6.971 | 10.827 | 19.415 | 40.029 | 85.088 |
| Latency StdDev (ms) | 3.032 | 3.763 | 4.821 | 6.258 | 12.960 | 35.641 | 90.246 |
| Transactions | 925112 | 1907871 | 2746453 | 3530102 | 3924979 | 3781089 | 3521583 |

{{< chart id="write_heavy_latency" >}}

Latency is increasing in exponentially. The latency growth rate is always increase, from 512 connections, double connection also double latency. That's understandable, maybe because when we double connections (more processes) in next scenario, they queue up for the resource. The latency is made up from the time processing and the time it waiting on the the queue.


{{< chart id="write_heavy_tps" >}}

TPS drop is increasing until c256, then drop. The growth rate of tps is always drop, from c16 to c32, double connection leads to double in tps, but after that, the growth rate reduce.
TPS drops and TPS growth rate also drops. It make me think that the resources like CPU has been doing something not useful when we increase the connection.

It make me think so, because, for example, there is a coffee shop with two bartenders, each can make one cup of coffee per minute, together they can make 2 cups per minute. If one people is waiting (no queue) they make 1 cup per minute (50% utilization). If two people request coffee (100% utilization) they make two cup per minute. if four people request (saturation) two people get serve and two people waiting in the queue, but they still making 2 cup per minute. No matter how many people are waiting, they consistently making two cups per minute. If they output drop, that mean the bartenders can't focus on their job and must doing something else like cleaning table.

#### Postgres metrics

##### pg_stat_activity raw wait event count
let's look at the stat activity to see those processes (connections) waiting on what

{{< tabs id="wait_event_count" >}}

---tab Broad view | top_wait_event_count_broad_view---

This broad view groups wait events by type. It is easier to read when the detailed chart has too many series.

---tab Detail view | top_wait_event_count---

cpu:running is not really a wait, it's mean the process is running on cpu, so this is a good thing.
client:clientread is also not need to care about. it's just that the client is slow or network is slow. the server wait for data from client. it doesn't cost cpu for these wait.

The more connection it have, the more lock it is waiting on. the total number of wait seem to double when we double connection

Lwlock:walwrite and lwlock:buffercontent are two most busiest lock. IO:WalSync also show up but without a clear pattern. LWLock:WalInsert shows up mostly at c1024.

Start from c128 and c256, we started to see: `IPC:ProcarrayGroupUpdate` and `LWLock:ProcArray`. These wait shows that there are overhead of connection managing when we increase the connection.

`lock:transactionid` show a clear pattern, when we double connection, it growth even faster.

minor wait events:
IO:DataFileExtend and lock:extend appeared rarely.
lock:tuple
timeout:vacuumdelay
timeout:spindelay

{{< /tabs >}}



##### pg_stat_activity wait event percents

{{< tabs id="wait_event_percent" >}}
---tab Broad view | top_wait_event_percent_broad_view---
- the cpu percent is reducing. That mean the more connections we add in, most of them result in waiting, only small portion are actually running.
- the io percent is also drop.

all other wait type show a increasing trend. that mean, the more connection we all, large portion of them are endup waiting for those locks.

---tab Detail view | top_wait_event_percent---
- Two dominant wait events are: LWLock:WalWrite and LWLock:BufferContent
{{< /tabs >}}


#### pg_stat_statement

**LWLock:BufferContent**

| Query | c16 | c32 | c64 | c128 | c256 | c512 | c1024 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| SELECT floor(balance)::int AS sender_balance<br>FROM wallets WHERE id = ? | — | — | — | — | — | — | 33.3% (47.12s) |
| SELECT id, balance<br>FROM wallets<br>WHERE id IN (? /*, ... */)<br>ORDER BY id<br>FOR UPDATE | — | — | — | — | 3.4% (26.12s) | — | 13.8% (1925.95s) |
| UPDATE wallets SET balance = balance - ?, updated_at = NOW() WHERE id = ? | — | — | — | — | — | 14.3% (57.83s) | 15.5% (142.65s) |
| INSERT INTO transactions (idempotency_key, type, amount, currency, status)<br>VALUES (gen_random_uuid(), ?, ?, ?, ?)<br>RETURNING id | — | — | — | 24.0% (191.50s) | 70.2% (1953.49s) | 85.7% (10433.48s) | 87.8% (41287.90s) |
| INSERT INTO ledger_entries (transaction_id, wallet_id, amount, direction)<br>VALUES<br>    (?, ?,   ?::numeric, ?),<br>    (?, ?, ?::numeric, ?) | — | 14.3% (68.47s) | 46.7% (457.26s) | 47.8% (1235.08s) | 85.6% (10810.96s) | 92.6% (43995.31s) | 89.5% (107084.30s) |
| UPDATE wallets SET balance = balance + ?, updated_at = NOW() WHERE id = ? | — | — | — | — | — | 18.2% (67.93s) | 55.3% (463.03s) |


**LWLock:WALWrite**

| Query | c16 | c32 | c64 | c128 | c256 | c512 | c1024 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| SELECT floor(balance)::int AS sender_balance<br>FROM wallets WHERE id = ? | — | — | — | — | — | 11.1% (8.44s) | 8.3% (11.78s) |
| SELECT id, balance<br>FROM wallets<br>WHERE id IN (? /*, ... */)<br>ORDER BY id<br>FOR UPDATE | — | — | — | — | — | 3.7% (119.11s) | 0.8% (116.72s) |
| UPDATE wallets SET balance = balance - ?, updated_at = NOW() WHERE id = ? | — | — | — | — | — | — | 2.8% (25.94s) |
| INSERT INTO transactions (idempotency_key, type, amount, currency, status)<br>VALUES (gen_random_uuid(), ?, ?, ?, ?)<br>RETURNING id | — | — | — | — | 1.0% (26.76s) | 4.9% (597.51s) | 2.8% (1318.48s) |
| INSERT INTO ledger_entries (transaction_id, wallet_id, amount, direction)<br>VALUES<br>    (?, ?,   ?::numeric, ?),<br>    (?, ?, ?::numeric, ?) | — | — | — | 2.2% (56.14s) | 3.5% (436.81s) | 3.1% (1475.25s) | 4.6% (5552.00s) |
| UPDATE wallets SET balance = balance + ?, updated_at = NOW() WHERE id = ? | — | — | — | — | 20.0% (49.09s) | 18.2% (67.93s) | 2.1% (17.81s) |

Both insert queries spend most of their time waiting on LWLock:BufferContent and LWLock:WALWrite.



After pg_stat_activity and pg_stat_statements, we can see that when we increase more connection, most of connection time now are waiting for. These waiting in theory should cost very little CPU, but how is it behave under high load? let's see some lower level metrics

#### OS Metrics

**PSI**
{{< tabs id="pressure_stall_information" >}}

---tab Broad view | psi_cpu_some---

huge pressure on cpu (no pressure on mem at all)

---tab Detail view | psi_io_some---

{{< /tabs >}}

relative low


## The conclusion
afs

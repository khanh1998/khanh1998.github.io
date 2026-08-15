---
date: '2026-04-19T11:58:31+07:00'
draft: false
title: 'Postgres Batch Insert Benchmark'
chartjs: true
---

I often hear that batch insert can help to increase the throughput. Instead of insert row by row, we can combine many rows into one batch and insert once. But I want to understand two things:
1. why is batching help increase insert throughput
2. if batching increase the throughput, why don't i just use a very huge batch. Is there a upper limit for a batch size.


## Benchmark setup

### Environment
- Sysbench running on EC2 `t3.micro` 2 vCPU, 1GB RAM
- Postgres 18 RDS `db.t4g.micro` 2 vCPU, 1GB RAM, 20GB storage, 90MB shared buffer
- Both EC2 and RDS are in the same region
- I choose Sysbench over PgBench because it help me to build the batch data from client with Lua script easily

### Scripts

#### Schema
```sql

-- unlogged table for faster seeding
CREATE UNLOGGED TABLE transactions (
    id             BIGSERIAL      PRIMARY KEY,
    account_id     BIGINT         NOT NULL,
    merchant_id    BIGINT         NOT NULL,
    amount         NUMERIC(12,2)  NOT NULL,
    currency       CHAR(3)        NOT NULL DEFAULT 'USD',
    status         SMALLINT       NOT NULL DEFAULT 0,
    type           SMALLINT       NOT NULL DEFAULT 0,
    reference_id   UUID           NOT NULL DEFAULT gen_random_uuid(),
    description    VARCHAR(255)   NOT NULL,
    ip_address     INET           NOT NULL,
    device_id      VARCHAR(64)    NOT NULL,
    metadata       JSONB          NOT NULL DEFAULT '{}',
    created_at     TIMESTAMPTZ    NOT NULL DEFAULT clock_timestamp(),
    updated_at     TIMESTAMPTZ    NOT NULL DEFAULT clock_timestamp()
);

-- seeding
INSERT INTO transactions (
    account_id, merchant_id, amount, currency, status, type,
    reference_id, description, ip_address, device_id, metadata,
    created_at, updated_at
)
SELECT
    (random() * {{NUM_ACCOUNTS}})::bigint,
    (random() * {{NUM_MERCHANTS}})::bigint,
    (random() * 10000)::numeric(12,2),
    (ARRAY['USD','EUR','GBP','AUD','SGD'])[(random()*4)::int + 1],
    (random() * 3)::smallint,
    (random() * 1)::smallint,
    gen_random_uuid(),
    'Payment ref-' || g,
    ('10.' || (random()*255)::int || '.' || (random()*255)::int || '.' || (random()*255)::int)::inet,
    'device-' || (random() * {{NUM_ACCOUNTS}})::bigint,
    jsonb_build_object('channel', (ARRAY['web','mobile','pos'])[(random()*2)::int + 1],
                       'attempt', (random()*3)::int + 1),
    NOW() - (random() * INTERVAL '90 days'),
    NOW() - (random() * INTERVAL '90 days')
FROM generate_series(1, {{NUM_ROWS}}) g;

ALTER TABLE transactions SET LOGGED;

CREATE INDEX ON transactions (account_id);
CREATE INDEX ON transactions (merchant_id);
CREATE INDEX ON transactions (account_id, created_at DESC);
CREATE INDEX ON transactions (status, created_at) WHERE status IN (0, 2);
CREATE INDEX ON transactions (created_at);

-- update table's statistic
VACUUM ANALYZE transactions;
CHECKPOINT;

-- warm the table
SELECT COUNT(*) FROM transactions;
```

#### Sysbench Lua script
```lua
-- batch_insert.lua
-- Sysbench Lua script for pg batch INSERT benchmark.
-- Accepts --batch-size=N on the CLI (passed via sysbench_options in mybench).
-- All row data is generated in the Lua VM (client side) before the query is sent.
--
-- Usage (standalone):
--   sysbench batch_insert.lua \
--     --pgsql-host=localhost --pgsql-port=5432 \
--     --pgsql-db=bench --pgsql-user=postgres \
--     --threads=8 --time=180 --batch-size=100 run

-- ---------------------------------------------------------------------------
-- Custom CLI options
-- ---------------------------------------------------------------------------
sysbench.cmdline.options = {
    batch_size = {"Number of rows to INSERT per transaction", 1}
}

-- ---------------------------------------------------------------------------
-- Per-thread setup / teardown
-- ---------------------------------------------------------------------------
local CURRENCIES = {"USD", "EUR", "GBP", "AUD", "SGD"}
local CHANNELS   = {"web", "mobile", "pos"}

function thread_init()
    drv = sysbench.sql.driver()
    con = drv:connect()
end

function thread_done()
    con:disconnect()
end

-- ---------------------------------------------------------------------------
-- Main benchmark event
-- Each call = one transaction inserting `batch_size` rows.
-- The entire VALUES list is built in Lua (client side) before the query fires.
-- ---------------------------------------------------------------------------
function event()
    local batch_size = tonumber(sysbench.opt.batch_size)
    local values     = {}

    for i = 1, batch_size do
        local account_id  = sysbench.rand.uniform(1,      100000)
        local merchant_id = sysbench.rand.uniform(1,       10000)
        local amount      = math.floor(sysbench.rand.uniform(1, 1000000)) / 100.0   -- 2 decimal places
        local currency    = CURRENCIES[sysbench.rand.uniform(1, #CURRENCIES)]
        local status      = sysbench.rand.uniform(0, 3)
        local txn_type    = sysbench.rand.uniform(0, 1)
        local channel     = CHANNELS[sysbench.rand.uniform(1, #CHANNELS)]
        local attempt     = sysbench.rand.uniform(1, 4)
        local device_id   = "device-" .. sysbench.rand.uniform(1, 100000)
        local ip          = sysbench.rand.uniform(0, 255) .. "." ..
                            sysbench.rand.uniform(0, 255) .. "." ..
                            sysbench.rand.uniform(0, 255) .. "." ..
                            sysbench.rand.uniform(1, 254)

        -- Escape single quotes in description just in case
        local description = "Payment ref-" .. sysbench.rand.uniform(1, 1000000)

        values[i] = string.format(
            -- account_id, merchant_id, amount, currency, status, type,
            -- reference_id, description, ip_address, device_id, metadata,
            -- created_at, updated_at
            "(%d, %d, %.2f, '%s', %d, %d, gen_random_uuid(), '%s', '%s'::inet, '%s', " ..
            "'{\"channel\":\"%s\",\"attempt\":%d}'::jsonb, clock_timestamp(), clock_timestamp())",
            account_id, merchant_id, amount, currency, status, txn_type,
            description, ip, device_id,
            channel, attempt
        )
    end

    local sql = "INSERT INTO transactions " ..
        "(account_id, merchant_id, amount, currency, status, type, " ..
        "reference_id, description, ip_address, device_id, metadata, " ..
        "created_at, updated_at) VALUES " ..
        table.concat(values, ",")

    con:query("BEGIN")
    con:query(sql)
    con:query("COMMIT")
end
```

### Run Parameters
- Number of threads: 2. Meaning two sysbench clients will concurrently send requests to RDS
- Duration: 180 seconds
- Batch size: 1, 10, 50, 100, 500, 1000, 2000, 5000, 10000
- Parameter value was used in the seeding file:
```
NUM_ACCOUNTS 100000
NUM_MERCHANTS 10000
NUM_ROWS 10000
```


Sample sysbench command:
```bash
sysbench --db-driver=pgsql --pgsql-host=localhost --pgsql-port=5432 --pgsql-user=postgres --pgsql-password=password --pgsql-db=benchmark --threads=2 --time=180 --batch-size=10000 --report-interval=5 ./benchmark.lua run
```

### Methodology
1. Choose a batch size (from small to large)
1. Drop the table if exist. Create the table and seeding some data.
2. Update statistic and request checkpoint.
3. Warm data.
4. Run sysbench benchmark. during the run, collect postgres metrics: pg_stat_activity, pg_stat_statement, pg_stat_tables,..., collect OS metrics: CPU, Disk, Memory,...
5. Drop the table
6. Wait for 30s and to the next benchmark with different batch size

All postgres metrics are collect in every 10 seconds and store to a timeseries database for analyze. OS metrics are collect every minute.

Note: since the EC2 and RDS instances are burstable, I only do benchmark when I got a alot of CPU and IO burstable credit, make sure it's won't ever run out during benchmark.

Weak points:
- All data are fit on share buffer
- for each batch size, benchmark is ran once, can suffer from outlier
- Since we use RDS, OS metrics from Enhance monitoring are not comprehensive
- `max_wal_size` of the instance is 2GB, `checkpoint_completion_target` is 0.9, checkpoint_timeout is 5 minutes . With the benchmark running in 180s, produce at most 2M row, it won't trigger checkpoint during benchmark.

## Result

### Overview

| Metric | batch_1 | batch_10 | batch_50 | batch_100 | batch_500 | batch_1000 | batch_2000 | batch_5000 | batch_10000 |
|---|---|---|---|---|---|---|---|---|---|
| TPS | 505.44 | 412.13 | 199.02 | 122.48 | 23.89 | 9.00 | 3.57 | 1.01 | 0.28 |
| QPS | 1516.31 | 1236.40 | 597.06 | 367.44 | 71.67 | 26.99 | 10.71 | 3.02 | 0.84 |
| Avg Latency (ms) | 3.95 | 4.85 | 10.05 | 16.33 | 83.70 | 222.31 | 560.33 | 1982.84 | 7135.48 |
| p95 Latency (ms) | 4.25 | 5.57 | 11.24 | 19.29 | 164.45 | 669.89 | 1903.57 | 6835.96 | 22034.77 |
| Transactions | 90981 | 74187 | 35825 | 22048 | 4301 | 1656 | 675 | 185 | 55 |
| Rows Inserted | 90,981 | 741,870 | 1,791,250 | 2,204,800 | 2,150,500 | 1,656,000 | 1,350,000 | 925,000 | 550,000 |

#### Rows Inserted vs Avg Latency

{{< chart id="transactions_latency" >}}

#### Avg Latency vs p95 Latency

{{< chart id="latency_p95" >}}

As you can see, both throughput (row inserted) and latency perform best at batch 100 for this specific workload and environment. Increase batch size larger than 100 barely help. That mean, for each workload and environment, there is a batch size that work best.

### Average Active Session (AAS)
During the benchmark run, my tool my take a snapshot of `pg_stat_activity` every 10s and store as a timeseries. In this analysis, we will care about the column `wait_event` and `wait_event_type`. Those two can let's we know what are the transaction are waiting on.

The query to compute the AAS table bellow are look like this:
```sql
SELECT
  COALESCE(wait_event_type, 'CPU') AS wait_event_type,
  COALESCE(wait_event, 'running')  AS wait_event,
  COUNT(*)                          AS occurrences,
  COUNT(DISTINCT _collected_at)     AS snapshot_count,
  COUNT(*)/COUNT(DISTINCT _collected_at) AS aas
FROM snap_pg_stat_activity
WHERE _run_id = ?
  AND state = 'active'
GROUP BY 1, 2
ORDER BY 3 DESC
LIMIT 20
```

For example, during benchmark 180s, we take a snapshot every 10s, there are 18 snapshots. the wait `IO:DataFileRead` occurs 2 times in 2 snapshot, it's AAS is 2/2 = 1.

#### WalSync and WalWrite lock explain
In this benchmark, we will see WalSync and WalWrite quite often. So I want to introduce about it.

In Postgres, there is a Wal buffer (4MB default). During the time data is sync to storage (long), the running transaction will write data to this buffer, and waiting for the next sync. This is a way to batch commits together and called group commit. LWLock:WalWrite and LWLock:WalSync is to protect this buffer.
When the transaction commit, it will call

| Wait Event | Description |
| ---------- | ----------- |
| LWLock:WalInsert | Waiting to insert WAL data into a memory buffer. |
| LWLock:WalWrite | Waiting for WAL buffers to be written to disk. |
| IO:WalWrite | Waiting for a write to a WAL file. |
| IO:WalSync | Waiting for a WAL file to reach durable storage. |

**LWLock:WalInsert**
This is a set of 8 locks protects the in memory WAL ring buffer. When a transaction want to write WAL record to WAL buffer. 8 locks meaning maximum 8 transactions can concurrently insert to wal buffer.
1. Reserve space in Wal buffer. No lock, CurrBytePos: [atomic fetch-and-add]
2. Copy data to reserved range in wal buffer (need one of 8 locks)

**IO:WalWrite** the `write` syscall
The first transaction (leader) in group commit will write wal buffer to os page cache.

**IO:WalSync** the `fdatasync` syscall
The leader in group commit wait for the OS to confirm data has physically reach the storage.

**LWLock:WalWrite**
The leader in group commit will take responsibility to write data to storage. the followers in group commit will wait on this.

```
All backends (parallel, competing)
    └── LWLock:WalInsert  →  write own record to WAL buffer

Flush point reached
    ├── One backend becomes leader
    │       └── IO:WalWrite → IO:WalSync → signals completion
    │
    └── Everyone else becomes followers
            └── LWLock:WalWrite  (waiting for leader's signal)

Leader signals → followers wake up → all proceed to commit
```

#### Other waits

**CPU:running**
This isn't wait, this mean the backend is running with cpu, not waiting on anything.
But it's not good if this is higher than vCPU. It indicating that the CPU is overload.

**Client:ClientRead**
Waiting to read data from the client.
This mean the client is slow and can't keep up with the speed of postgres. There can be because of slow network between client and postgres.

**IO:DataFileRead**
Waiting for a read from a relation data file.

**Timeout:VacuumDelay**
Waiting in a cost-based vacuum delay point.

**IO:DataFileExtend**
Waiting for a relation data file to be extended. Tables and indexes in Postgres are files under the hood. When there are no free pages left in the files, postgres must extend the files by writing empty pages to it.

**Lock:extend**
Waiting to extend a relation. only one relation can extend at a time.

**LWLock:BufferContent**
Waiting to access a data page in memory. Each 8KB pages in shared buffer has a internal latch (light weight lock), transaction when access these page need to acquire this lock.

**IO:WalInitWrite**
Waiting for a write while initializing a new WAL file.

#### batch_1

| Wait Type | Wait Event | Count | Load (AAS/2 vCPU) |
|---|---|---|---|
| CPU | running | 19 | 1.06 |
| IO | WalSync | 12 | 1.00 |
| LWLock | WALWrite | 1 | 1.00 |

From the batch 1, we already see pressure on the WAL because of this write heavy workload.

#### batch_10

| Wait Type | Wait Event | Count | Load (AAS/2 vCPU) |
|---|---|---|---|
| CPU | running | 26 | 1.44 |
| Client | ClientRead | 7 | 1.17 |
| IO | WalSync | 6 | 1.00 |
| LWLock | WALWrite | 1 | 1.00 |

#### batch_50

| Wait Type | Wait Event | Count | Load (AAS/2 vCPU) |
|---|---|---|---|
| CPU | running | 33 | 1.74 |
| Client | ClientRead | 7 | 1.17 |
| IO | WalSync | 4 | 1.00 |
| IO | DataFileRead | 2 | 1.00 |
| IO | WalInitWrite | 1 | 1.00 |
| LWLock | WALWrite | 1 | 1.00 |
| Timeout | VacuumDelay | 1 | 1.00 |

Until batch 50, the postgres is still under utilization AAS < 2 vCPU baseline.

#### batch_100

| Wait Type | Wait Event | Count | Load (AAS/2 vCPU) |
|---|---|---|---|
| CPU | running | 40 | 2.22 |
| Client | ClientRead | 3 | 1.00 |
| Lock | extend | 2 | 2.00 |
| Timeout | VacuumDelay | 2 | 2.00 |
| IO | DataFileExtend | 1 | 1.00 |
| IO | DataFileRead | 1 | 1.00 |
| LWLock | BufferContent | 1 | 1.00 |

The CPU is overload here.

#### batch_500

| Wait Type | Wait Event | Count | Load (AAS/2 vCPU) |
|---|---|---|---|
| CPU | running | 58 | 3.05 |
| Client | ClientRead | 2 | 1.00 |
| Timeout | VacuumDelay | 1 | 1.00 |

#### batch_1000

| Wait Type | Wait Event | Count | Load (AAS/2 vCPU) |
|---|---|---|---|
| CPU | running | 71 | 3.74 |
| Client | ClientRead | 1 | 1.00 |
| LWLock | BufferContent | 1 | 1.00 |

#### batch_2000

| Wait Type | Wait Event | Count | Load (AAS/2 vCPU) |
|---|---|---|---|
| CPU | running | 68 | 3.58 |
| Client | ClientRead | 1 | 1.00 |
| LWLock | BufferContent | 1 | 1.00 |
| Timeout | VacuumDelay | 1 | 1.00 |

#### batch_5000

| Wait Type | Wait Event | Count | Load (AAS/2 vCPU) |
|---|---|---|---|
| CPU | running | 65 | 3.42 |
| Timeout | VacuumDelay | 3 | 1.00 |
| LWLock | BufferContent | 1 | 1.00 |

#### batch_10000

| Wait Type | Wait Event | Count | Load (AAS/2 vCPU) |
|---|---|---|---|
| CPU | running | 72 | 3.60 |
| LWLock | BufferContent | 2 | 1.00 |
| Client | ClientRead | 1 | 1.00 |
| IO | DataFileExtend | 1 | 1.00 |

### OS metrics
peak cpu
batch_1
19.4%
batch_10
25.6%
batch_50
42.5%
batch_100
48.5%
batch_500
51.6%
batch_1000
45.7%
batch_2000
42.3%
batch_5000
41.6%
batch_10000
36.3%


## Unnest insert

### Overview

| Metric | batch_1 | batch_10 | batch_50 | batch_100 | batch_500 | batch_1000 | batch_2000 | batch_5000 | batch_10000 |
|---|---|---|---|---|---|---|---|---|---|
| TPS | 473.85 | 421.61 | 223.23 | 118.58 | 24.46 | 12.18 | 6.05 | 2.24 | 0.38 |
| QPS | 1421.54 | 1264.82 | 669.70 | 355.74 | 73.39 | 36.53 | 18.14 | 6.72 | 1.15 |
| Avg Latency (ms) | 4.22 | 4.74 | 8.96 | 16.86 | 81.73 | 163.95 | 330.71 | 891.89 | 5229.24 |
| p95 Latency (ms) | 4.91 | 5.77 | 16.12 | 38.94 | 383.33 | 669.89 | 1235.62 | 2828.87 | 21255.35 |
| Transactions | 85296 | 75892 | 40187 | 21346 | 4409 | 2200 | 1091 | 406 | 78 |
| Rows Inserted | 85,296 | 758,920 | 2,009,350 | 2,134,600 | 2,204,500 | 2,200,000 | 2,182,000 | 2,030,000 | 780,000 |

#### Rows Inserted vs Avg Latency

{{< chart id="unnest_rows_latency" >}}

#### Avg Latency vs p95 Latency

{{< chart id="unnest_latency_p95" >}}

#### CPU

{{< chart id="unnest_cpu" >}}

#### IO

##### Write IOPS vs Read IOPS

{{< chart id="unnest_iops" >}}

##### Disk Queue Length vs Await

{{< chart id="unnest_disk" >}}

##### Utilization
+1m 0s
batch_10000: 98.60
batch_5000: 87.96
batch_2000: 29.97
batch_500: 23.78
batch_1000: 22.32
batch_100: 20.70
batch_50: 14.98
batch_10: 6.67
batch_1: 1.28

### Load Average:
+1m 0s
batch_10000: 14.23
batch_5000: 10.28
batch_1000: 3.08
batch_2000: 2.54
batch_500: 2.53
batch_50: 2.26
batch_100: 1.30
batch_10: 1.08
batch_1: 0.67

## Comparison

### Rows Inserted

{{< chart id="compare_rows_inserted" >}}

### Avg Latency

{{< chart id="compare_avg_latency" >}}


### Sysbench Lua script

```lua
-- batch_insert_unnest.lua
-- Sysbench Lua script for pg batch INSERT benchmark using unnest().
-- Instead of a flat VALUES list, builds one array literal per column and uses:
--   INSERT … SELECT … FROM unnest(ARRAY[…]::type[], …) AS t(col, …)
-- This produces a single, fixed-shape query regardless of batch size, which
-- can benefit from plan caching and may reduce per-row parse overhead.
--
-- Accepts --batch-size=N on the CLI (passed via sysbench_options in mybench).

sysbench.cmdline.options = {
    batch_size = {"Number of rows to INSERT per transaction", 1}
}

local CURRENCIES = {"USD", "EUR", "GBP", "AUD", "SGD"}
local CHANNELS   = {"web", "mobile", "pos"}

function thread_init()
    drv = sysbench.sql.driver()
    con = drv:connect()
end

function thread_done()
    con:disconnect()
end

function event()
    local batch_size = tonumber(sysbench.opt.batch_size)

    -- Per-column arrays (avoids one big table of tuples)
    local account_ids   = {}
    local merchant_ids  = {}
    local amounts       = {}
    local currencies    = {}
    local statuses      = {}
    local txn_types     = {}
    local descriptions  = {}
    local ip_addresses  = {}
    local device_ids    = {}
    local channels      = {}
    local attempts      = {}

    for i = 1, batch_size do
        account_ids[i]  = sysbench.rand.uniform(1, 100000)
        merchant_ids[i] = sysbench.rand.uniform(1, 10000)
        amounts[i]      = string.format("%.2f", math.floor(sysbench.rand.uniform(1, 1000000)) / 100.0)
        currencies[i]   = CURRENCIES[sysbench.rand.uniform(1, #CURRENCIES)]
        statuses[i]     = sysbench.rand.uniform(0, 3)
        txn_types[i]    = sysbench.rand.uniform(0, 1)
        descriptions[i] = "Payment ref-" .. sysbench.rand.uniform(1, 1000000)
        ip_addresses[i] = sysbench.rand.uniform(0, 255) .. "." ..
                          sysbench.rand.uniform(0, 255) .. "." ..
                          sysbench.rand.uniform(0, 255) .. "." ..
                          sysbench.rand.uniform(1, 254)
        device_ids[i]   = "device-" .. sysbench.rand.uniform(1, 100000)
        channels[i]     = CHANNELS[sysbench.rand.uniform(1, #CHANNELS)]
        attempts[i]     = sysbench.rand.uniform(1, 4)
    end

    -- Build array literals for each column
    local function int_array(t)
        return "ARRAY[" .. table.concat(t, ",") .. "]"
    end
    local function quoted_array(t)
        local q = {}
        for i, v in ipairs(t) do q[i] = "'" .. v .. "'" end
        return "ARRAY[" .. table.concat(q, ",") .. "]"
    end
    local function numeric_array(t)
        return "ARRAY[" .. table.concat(t, ",") .. "]"
    end

    -- metadata is built from channels + attempts
    local metadata_arr = {}
    for i = 1, batch_size do
        metadata_arr[i] = string.format("'{\"channel\":\"%s\",\"attempt\":%d}'",
            channels[i], attempts[i])
    end

    local sql = string.format([[
INSERT INTO transactions
    (account_id, merchant_id, amount, currency, status, type,
     reference_id, description, ip_address, device_id, metadata,
     created_at, updated_at)
SELECT
    a, m, n::numeric(12,2), c, s::smallint, tp::smallint,
    gen_random_uuid(), d, ip::inet, dev, meta::jsonb,
    clock_timestamp(), clock_timestamp()
FROM unnest(
    %s::bigint[],
    %s::bigint[],
    %s::numeric[],
    %s::text[],
    %s::int[],
    %s::int[],
    %s::text[],
    %s::text[],
    %s::text[],
    %s::jsonb[]
) AS t(a, m, n, c, s, tp, d, ip, dev, meta)]],
        int_array(account_ids),
        int_array(merchant_ids),
        numeric_array(amounts),
        quoted_array(currencies),
        int_array(statuses),
        int_array(txn_types),
        quoted_array(descriptions),
        quoted_array(ip_addresses),
        quoted_array(device_ids),
        -- jsonb array: elements are already single-quoted objects
        "ARRAY[" .. table.concat(metadata_arr, ",") .. "]"
    )

    con:query("BEGIN")
    con:query(sql)
    con:query("COMMIT")
end
```

### AAS
#### batch_1

| Wait Type | Wait Event | Count | Load (AAS/4 vCPU) |
|---|---|---|---|
| CPU | running | 25 | 1.39 |
| IO | WalSync | 9 | 1.00 |

#### batch_10

| Wait Type | Wait Event | Count | Load (AAS/4 vCPU) |
|---|---|---|---|
| CPU | running | 28 | 1.56 |
| IO | WalSync | 9 | 1.00 |
| LWLock | WALWrite | 1 | 1.00 |

#### batch_50

| Wait Type | Wait Event | Count | Load (AAS/4 vCPU) |
|---|---|---|---|
| CPU | running | 24 | 1.33 |
| IO | DataFileRead | 11 | 1.57 |
| IO | WalSync | 7 | 1.00 |
| LWLock | WALWrite | 3 | 1.00 |
| Timeout | VacuumDelay | 3 | 1.00 |
| Client | ClientRead | 2 | 2.00 |
| IO | WalInitWrite | 2 | 1.00 |
| IO | DataFileWrite | 1 | 1.00 |

#### batch_100

| Wait Type | Wait Event | Count | Load (AAS/4 vCPU) |
|---|---|---|---|
| CPU | running | 28 | 1.56 |
| IO | DataFileRead | 21 | 1.75 |
| IO | WalSync | 5 | 1.00 |
| Timeout | VacuumDelay | 3 | 1.00 |
| LWLock | WALWrite | 2 | 1.00 |

#### batch_500

| Wait Type | Wait Event | Count | Load (AAS/4 vCPU) |
|---|---|---|---|
| IO | DataFileRead | 29 | 1.93 |
| CPU | running | 23 | 1.28 |
| Timeout | VacuumDelay | 6 | 1.00 |
| Client | ClientRead | 2 | 1.00 |
| IO | DataFilePrefetch | 2 | 1.00 |
| IO | WalSync | 1 | 1.00 |

#### batch_1000

| Wait Type | Wait Event | Count | Load (AAS/4 vCPU) |
|---|---|---|---|
| IO | DataFileRead | 27 | 1.80 |
| CPU | running | 26 | 1.44 |
| Timeout | VacuumDelay | 4 | 1.00 |
| IO | WalSync | 3 | 1.00 |
| IO | DataFilePrefetch | 1 | 1.00 |
| IO | DataFileWrite | 1 | 1.00 |

#### batch_2000

| Wait Type | Wait Event | Count | Load (AAS/4 vCPU) |
|---|---|---|---|
| IO | DataFileRead | 30 | 2.00 |
| CPU | running | 26 | 1.44 |
| Timeout | VacuumDelay | 5 | 1.00 |
| LWLock | BufferContent | 2 | 1.00 |
| IO | WalSync | 1 | 1.00 |

#### batch_5000

| Wait Type | Wait Event | Count | Load (AAS/4 vCPU) |
|---|---|---|---|
| CPU | running | 29 | 1.53 |
| IO | DataFileRead | 28 | 2.00 |
| Client | ClientRead | 2 | 1.00 |
| IO | WalWrite | 1 | 1.00 |

#### batch_10000

| Wait Type | Wait Event | Count | Load (AAS/4 vCPU) |
|---|---|---|---|
| CPU | running | 78 | 3.71 |
| Client | ClientRead | 3 | 1.00 |


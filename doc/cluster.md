Building cluster
================

Replication
-----------

- Configure clickhouse, see [doc](https://clickhouse.yandex/reference_en.html#Data_replication) for more details.
- Create Replicated tables

```sql
CREATE DATABASE graphite;

CREATE TABLE graphite.metrics
(
    date Date DEFAULT toDate(0),
    name String,
    level UInt16,
    parent String,
    updated DateTime DEFAULT now(),
    status Enum8('SIMPLE' = 0, 'BAN' = 1, 'APPROVED' = 2, 'HIDDEN' = 3, 'AUTO_HIDDEN' = 4)
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/single/graphite.metrics', '{replica}', updated)
PARTITION BY toYYYYMM(date)
ORDER BY (parent, name)
SETTINGS index_granularity = 1024;

CREATE TABLE graphite.data_lr
(
    metric String,
    value Float64,
    timestamp UInt32,
    date Date,
    updated UInt32
)
ENGINE = ReplicatedGraphiteMergeTree('/clickhouse/tables/graphite.data', '{replica}', 'graphite_rollup')
PARTITION BY toMonday(date) -- or, if you have low amount of data, then toYYYYMM(date)
ORDER BY (metric, timestamp)
SETTINGS index_granularity = 8192;
```

Sharding and Replication
------------------------

- Configure clickhouse, see [replicating](https://clickhouse.yandex/reference_en.html#Data_replication) and [sharding](https://clickhouse.yandex/reference_en.html#Distributed) docs for more details.
- Create tables

```sql
CREATE DATABASE graphite;

CREATE TABLE graphite.metrics
(
    date Date DEFAULT toDate(0),
    name String,
    level UInt16,
    parent String,
    updated DateTime DEFAULT now(),
    status Enum8('SIMPLE' = 0, 'BAN' = 1, 'APPROVED' = 2, 'HIDDEN' = 3, 'AUTO_HIDDEN' = 4)
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/single/graphite.metrics', '{replica}', updated)
PARTITION BY toYYYYMM(date)
ORDER BY (parent, name)
SETTINGS index_granularity = 1024;

CREATE TABLE graphite.data_lr
(
    metric String,
    value Float64,
    timestamp UInt32,
    date Date,
    updated UInt32
)
ENGINE = ReplicatedGraphiteMergeTree('/clickhouse/tables/{shard}/graphite.data_lr', '{replica}', 'graphite_rollup')
PARTITION BY toMonday(date) -- or, if you have low amount of data, then toYYYYMM(date)
ORDER BY (metric, timestamp)
SETTINGS index_granularity = 8192;


CREATE TABLE graphite.data
(
    metric String,
    value Float64,
    timestamp UInt32,
    date Date,
    updated UInt32
)
ENGINE Distributed(CLICKHOUSE_CLUSTER_NAME, 'graphite', 'data_lr', sipHash64(metric));
```

Don't forget to replace ```CLICKHOUSE_CLUSTER_NAME``` with name from you config.

**Notice**: We use sharding only for ```data```, couse ```metrics``` is small and contains only metric names.

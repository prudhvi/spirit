# What is this?

Spirit is a _clone_ of the schema change tool [gh-ost](https://github.com/github/gh-ost).

It works very similar to gh-ost except:
- It only supports MySQL 5.7 and MySQL 8.0 (it is anticipated only 8.0 will be supported once Aurora v2 reaches end of life)
- It is multi-threaded in both the row-copying and the binlog applying phase
- It supports resume-from-checkpoint

The goal of spirit is to apply schema changes much faster than gh-ost. This makes it unsuitable in the following scenarios:
- You use read-replicas to serve traffic
- You have tables with `VARCHAR` primary keys
- You require support for older versions of MySQL

If this is the case, `gh-ost` remains a fine choice.

See [USAGE](USAGE.md) for more information on how to use spirit.

## Optimizations

The following are some of the optimizations that make spirit faster than gh-ost:

### Dynamic Chunking

Rather than accept a fixed chunk size (such as 1000 rows), spirit instead takes a target chunk time (such as 500ms). It then dynamically adjusts the chunk size to meet this target. This is both safer for very wide tables with a lot of indexes and faster for smaller tables.

500ms is quite "high" for traditional MySQL environments, but remember _spirit does not support read-replicas_. This helps it copy chunks as efficiently as possible.

### Ignore Key Above Watermark

As spirit is copying rows, it keeps track of the highest key-value that either has been copied, or could be in the process of being copied. This is called the "high watermark". As rows are discovered from the binary log, they can be discarded if the key is above the high watermark. This is because once the copier reaches this point, it is guaranteed it will copy the latest version of the row.

In practice, this optimization works really well when your table has an `auto_increment` `PRIMARY KEY` and most of the inserts or modifications are at the end of the table.

**Note:** Spirit does not support `VARCHAR` primary keys, so it does not need to worry about collation issues when comparing if a key is above another key.

### Change Row Map

As spirit discovers rows that have been changed via the binary log, it stores them in a map. Or rather, it stores the key, and if the last operation was a `DELETE` or any other operation. This is called the "change row map". Periodically it then flushes the change row map by batching a large `REPLACE INTO new_table .. SELECT FROM old_table` and `DELETE FROM new_table WHERE pk IN (..)` statement.

In some workloads this can result in significant performance improvements, because updates from the binary log are merged and de-duplicated. i.e. if a row is updated 10 times, it will only be copied once.

**Note:** Spirit does not support `VARCHAR` primary keys, so it does not need to worry about collations where 'a' != 'A' in the change row map.

### Multi-threaded copy

Spirit will copy rows in multiple threads. This optimization really requires MySQL 8.0+ to make sense, which has much better support for multi-threaded replication.

While spirit does not support read-replicas, it still tries to keep replication mostly up to date (with support for reading a replica every 2 seconds and observing lag). The replication monitor is not intended to be as high fidelity as gh-ost, and only used to ensure that DR functionality is not impacted.

### Attempt Instant DDL

Spirit will attempt to use MySQL 8.0's `INSTANT` DDL assertion before applying the change itself. If the DDL change supports it, `INSTANT DDL` is a very fast operation and only requires a metadata change.

**Note:** This feature has been contributed to `gh-ost` by the same authors of Spirit. It is disabled by default, and [only in the master branch](https://github.com/github/gh-ost/blob/master/doc/command-line-flags.md#attempt-instant-ddl).

## Performance

Our internal goal for Spirit is to be able to migrate a 10TiB table in under 5 days. We believe we are able to achieve this in most-cases, but it depends on:
- The version of MySQL used (5.7 is much worse due `innodb_autoinc_lock_mode=1` being the default).
- How many secondary indexes the table has.
- How many active changes are being made to the table.
- The `threads` and `target-chunk-time` that is used.
- If any replication throttler is used.
- If the MySQL server becomes significantly IO bound (at this point, the migration might slow down a lot)

For proof that it is possible, here is the final output from a migration on a 10TiB `finch.xfers` table on Aurora v3:

```
time="2023-04-21T07:08:24Z" level=info msg="apply complete: instant-ddl=false inplace-ddl=false total-chunks=926661 copy-rows-time=59h27m9.285730804s checksum-time=6h11m2.244079686s total-time=65h38m12.790047338s"
```

This table does [include some secondary indexes](https://github.com/square/finch/blob/65fef3da97cfb24892ef283bc93ab8f09c4fb732/test/workload/xfer/schema.sql#L39-L62), but the table was idle and no replication throttler was used. The configuration used `threads=8` and `target-chunk-time=2s`, which is on the higher end of normal. We attempted to run a comparison with gh-ost (w/a 10K chunk-size), but canceled it after 10 days.

For a non-idle table, the performance delta is even greater. Consider the following microbench performed on a m1 mac with 10 cores and MySQL 8.0.31 using defaults:

| Table/Scenario                               | Gh-ost   | spirit  | spirit (checksum disabled) |
| -------------------------------------------- | -------- | ------- | -------------------------- |
| finch.balances (800MB/1M rows), idle load    | 28.720s  | 11.197s | 9.278s                     |
| finch.balances (800MB/1M rows), during bench | 2:50m+   | ~15-18s | ~15-18s                    |

This scenario is kind of a worse case for gh-ost since it prioritizes replication over row-copying and the benchmark never lets up. The checksum feature is not present in gh-ost, and adds about 10-20% to migration time. We typically do not recommend disabling it.

## Unsupported Features

- **`RENAME` column**. Spirit only supports `RENAME` if it applies via the `INSTANT` DDL algorithm (MySQL 8.0+). This might mean that you need to break up some schema changes to perform the `RENAME` operations first, and then the non-`INSTANT` DDL changes after. From a code perspective: rename is tricky to add support for, because the copier can no longer take a simple intersection of columns between the old-and-new table. If you consider more complex DDLs that include a `RENAME` and an `ADD COLUMN` (i.e. `RENAME COLUMN c1 TO n1, ADD COLUMN c1 varchar(100)`) it's easy to get these wrong, leading to data corruption. This is why we do not intend to support this feature.
- **`VARCHAR` PRIMARY KEY**. Spirit does not support `VARCHAR` primary keys. This is because it has optimizations that do not work safely with collations. If you want to use a `uuid` primary key, make sure you define it as `VARBINARY` instead. From a code perspective: the "ignore key above watermark" optimization is easy to disable, but the "change row map" is not. If Spirit was to support `VARCHAR` primary keys, it would require special handling of the binary log changes to apply in the correct order, making it a non-trivial change.
- **`ALTER` PRIMARY KEY**. Spirit requires the table to have a primary key, and the primary key can not be altered by the schema change. There might be some flexibility to support UNIQUE keys and some modifications of the primary key in future, but it is not a priority for now.

## Risks and Limitations

Writing a new data migration tool is scary, since bugs have real consequences (data loss). These are the main problems we anticipate:

1. The cut-over algorithm is not as battle tested as gh-ost's. We have studied it in detail, and believe it's comparable. We believe we've managed to mitigate the other risks up until cut-over with the introduction of a checksum feature, which is enabled by default.
2. Spirit does not support as many different table types as gh-ost. Currently, primary keys can be int/bigint \[unsigned\] or varbinary. Composite primary keys are still supported, but there are currently no plans to support `VARCHAR` primary keys.
3. We have tried to balance making Spirit _as fast as possible_ while still being safe to run on production systems that are running existing workloads. Sometimes this means spirit might venture into creating slow downs in application performance. If it does, please file an issue and help us make improvements.
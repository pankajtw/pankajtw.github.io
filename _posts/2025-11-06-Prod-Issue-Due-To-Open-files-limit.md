---
title: "Fixing Prod Query Slowdowns Caused by MySQL open_files_limit"
date: 2025-11-03
categories: [MySQL, Performance Tuning, RCA]
tags: [mysql, table_open_cache, open_files_limit, performance, pmm, production-issue]
description: "A real-world production incident where a small table_open_cache caused severe query slowdowns, how we diagnosed it, and what fixed it."
---

## 🧩 The Problem: Queries Stuck in “Opening/Closing Tables”

During peak traffic hours on our **production MySQL primary** (`prod-shard3-db01`), we started observing a sudden spike in query latency across multiple application services.  
`SHOW PROCESSLIST` revealed that **most active queries** were in the states:

```bash
|Opening tables | SELECT ... FROM tbl_abc ...... | 
Opening tables | SELECT ... FROM tbl_lmn ...... |
Closing tables | SELECT ... FROM tbl_xyz ...
```

and they were staying there unusually long — in some cases over several seconds.  
Application performance degraded sharply, with slow responses and intermittent timeouts. 

## 🔍 First Observation: Table Cache Thrashing

Our PMM dashboard “MySQL Table Open Cache Status” confirmed the suspicion —  
the **Table Open Cache hit ratio dropped to 10%**, while **misses due to overflows** spiked sharply (~60K ops/sec).

![table cache misses](/assests/images/table_cache_misses.png)

In simple terms, MySQL was unable to reuse already-opened tables because its **table_open_cache** was **too small**.  
As a result, every query was **opening and closing table descriptors** repeatedly — a heavy overhead, especially under thousands of concurrent connections.

## ⚙️ Step 1 — First Fix: Increase `table_open_cache` from 400 → 3000

We initially noticed that the server had the following value:

```sql
mysql> SHOW GLOBAL VARIABLES LIKE 'table_open_cache';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| table_open_cache   | 400   |
+--------------------+-------+
```

We increased it to 3000:

```sql
SET PERSIST table_open_cache = 3000;
```

Almost immediately, the “Opening tables” entries in the processlist began to disappear and most queries started executing normally again.

However, in PMM we could still see cache misses and a few overflow spikes, indicating that MySQL was still occasionally running out of cached table descriptors during high load.

## ⚙️ Step 2 — Final Fix: Increase table_open_cache to 8192

We further bumped the value to 8192:

```sql
SET PERSIST table_open_cache = 8192;
```

This completely stabilized the cache behavior — the Table Open Cache Hit Ratio went to 100%,
and misses and overflows dropped to zero.

![table cache hits](/assests/images/table_cache_hits.png)

Application latency normalized immediately, and the MySQL Opened_tables status variable remained steady verfied from 

```sql
show global status like 'opened_tables'
```

🧐 But Why Was table_open_cache So Small Initially?

Here’s where it got interesting.

When we originally had attempted to configure in the my.cnf file below values few weeks back:

```bash
table_open_cache = 40000
max_connections  = 10000
```

MySQL refussed it and logged below warnings in the error log 

```bash
2025-10-01T20:38:20.318797Z 0 [Warning] [MY-010140] [Server] Could not increase number of max_open_files to more than 10000 (request: 386641)
2025-10-01T20:38:20.318801Z 0 [Warning] [MY-010141] [Server] Changed limits: max_connections: 9190 (requested 10000)
2025-10-01T20:38:20.318804Z 0 [Warning] [MY-010142] [Server] Changed limits: table_open_cache: 400 (requested 40000)
```

The root cause?

The OS-level open file limit for the MySQL service (mysqld) was capped at 10,000.
Since each table and connection consumes file descriptors, MySQL could not honor the higher cache and connection settings, and it automatically reduced table_open_cache down to 400.

```bash
root@prod-shard3-db01:/home/pankaj# pid=$(pidof mysqld)
root@prod-shard3-db01:/home/pankaj# cat /proc/$pid/limits | awk 'NR==1 || /open files/'
Limit                     Soft Limit           Hard Limit           Units     
Max open files            10000                10000                files
```

## ⚠️ Discovery: The “LimitNOFILE=Infinity” That Never Took Effect

While reviewing configuration, we found that a systemd override had already been created at:

```bash
/etc/systemd/system/mysql.service.d/override.conf
```
with the content:
```bash
[Service]
LimitNOFILE=Infinity
```

At first glance, this should have allowed MySQL to open unlimited file descriptors. That’s when we realized the catch:
the MySQL service was never restarted after adding the override file.
The LimitNOFILE=Infinity directive was not yet applied to the running systemd service. We verfied this by comparing mysql uptime (mysqladmin status), with the commit for the change made to the override file.

## 🔄 Step 3 — Restart to Apply the Systemd Override

After restarting the MySQL service during a maintenance window:

```bash
systemctl restart mysql
```

The new limit took effect successfully:

```bash
cat /proc/$(pidof mysqld)/limits | grep "open files"
Max open files: 1048576
```

Now the OS-level limit comfortably supported our desired table_open_cache and max_connections values.

## 🧠 Summary

This incident reinforced how MySQL performance tuning is as much about the OS layer as it is about MySQL itself.

A small table_open_cache combined with a tight open_files_limit caused thousands of queries to waste time repeatedly opening and closing tables.
A few simple configuration adjustments — increasing table_open_cache and correctly applying the LimitNOFILE=Infinity override — brought the system back to full health.



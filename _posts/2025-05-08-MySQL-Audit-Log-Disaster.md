---
title: "When a MySQL Replica Became Primary and Blew Up the Disk with Audit Logs"
date: 2025-05-09
tags: [mysql, percona, audit-log, replication]
---


# 🚨 When a MySQL Replica Became Primary and Blew Up the Disk with Audit Logs

During a scheduled maintenance window on a Saturday, I promoted one of our MySQL replicas to act as the new **primary**. Everything went smoothly — replication was caught up, `read_only` was disabled, writes were flowing in, and the app stayed healthy.

But come Monday morning, we were greeted with a nasty surprise:

> **Disk space alert 🔥: `/mnt/DATABASE` usage > 90%**

A quick investigation revealed the root cause:

> The **audit log file** had grown to **2.1TB** in just under 48 hours!

---

## 💣 What Went Wrong

The promoted replica had **Percona’s Audit Log plugin** enabled with the following configuration:

```sql
audit_log_policy = ALL
audit_log_handler = FILE
audit_log_rotate_on_size = 0
audit_log_strategy = ASYNCHRONOUS
```

Let me break this down:

| Setting | Meaning |
|--------|--------|
| `ALL` | Log everything — SELECTs, INSERTs, logins, DDLs |
| `FILE` | Write logs to a flat file (`audit.log`) in `datadir` |
| `rotate_on_size = 0` | No automatic log rotation — keep appending forever |
| `ASYCHRONOUS` | Non-blocking writes to the log (good!) |

As a **replica**, this configuration didn’t cause problems — replicas have minimal write traffic.  
But once promoted to **primary**, it started logging **every single query** from our read-heavy production workloads.

And since **rotation was disabled**, the log just kept growing... until it ate 2.1TB of disk space.

---

## 🔧 How We Fixed It (Without Restarting MySQL)

### ✅ 1. Confirmed the audit log file path

```bash
mysql -e "SELECT @@datadir;"
```

The actual file path turned out to be:

```bash
/mnt/DATABASE/mysql/audit.log
```

(with a symlink from `/var/lib/mysql/audit.log`)

---

### ✅ 2. Verified that MySQL was still writing to it

```bash
sudo lsof /mnt/DATABASE/mysql/audit.log
```

This showed that the `mysqld` process still had the file open, so we **couldn't delete it directly**.

---

### ✅ 3. Safely truncated the log file in place

```bash
sudo truncate -s 0 /mnt/DATABASE/mysql/audit.log
```

This instantly freed up 2.1TB without interrupting MySQL.

Since we were using `audit_log_strategy = ASYNCHRONOUS`, the server didn’t mind — it just kept writing into the now-empty file.

---

### ✅ 4. (Optional) Rotated logs manually

To force MySQL to start a new log file immediately:

```sql
FLUSH AUDIT_LOGS;
```

---

## 🛡️ How We Prevented This from Happening Again

### 🔁 Enabled audit log rotation:

```sql
SET GLOBAL audit_log_rotate_on_size = 1073741824;  -- Rotate at 1GB
SET GLOBAL audit_log_rotations = 5;               -- Keep 5 rotated logs
```

Added it permanently to `/etc/my.cnf`:

```ini
[mysqld]
audit_log_rotate_on_size = 1073741824
audit_log_rotations = 5
```

### 📉 Reduced audit verbosity (optional)

If not all queries need to be logged, consider switching:

```sql
SET GLOBAL audit_log_policy = LOGINS;  -- Only log login events
```

---

## 🧠 Takeaways

This incident was a great reminder that **promoting a replica to primary is not just a replication config change** — it comes with a shift in **operational responsibilities**, including:

- Disk I/O
- Query volume
- Log generation
- Backup behaviors

### ✅ Always check:

| ✅ Checklist Before Promotion |
|-----------------------------|
| Are audit/slow/general logs enabled? |
| Are there hardcoded log paths or retention gaps? |
| Will cron jobs (like backups or maintenance) still behave correctly? |
| Is disk usage being monitored and alerted proactively? |

---

## 🧰 Bonus: Truncate Log File via Cron (Failsafe)

Until you're confident with log rotation, this cron can be your insurance:

```bash
0 3 * * * /usr/bin/truncate -s 0 /mnt/DATABASE/mysql/audit.log
```

---

## 💬 Final Thoughts

We averted a major production impact because we caught it early Monday — but in a different scenario, **2.1TB of audit logs could’ve taken the server offline**.

Always treat a replica promotion as a full-blown cutover — not just a `read_only = 0`.

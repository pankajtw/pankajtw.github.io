---
title: "How pt-heartbeat Log Rotation Caused Disk Space Bloat on Our MySQL Nodes"
date: 2025-10-10
categories: [MySQL, Percona, Monitoring, Linux]
tags: [pt-heartbeat, logrotate, systemd, file-descriptor, disk-space]
---

We recently faced a subtle but critical disk space bloat issue on one of our MySQL production nodes, and the root cause turned out to be how **Percona’s `pt-heartbeat`** handles logging during log rotation. This blog details the original setup, what went wrong, and the fix that worked for us.

---

## Original Setup: `pt-heartbeat` as a Systemd Service

We had `pt-heartbeat` running as a systemd service on both primary and replica MySQL servers to monitor replication lag.

On the **primary**, it runs in update mode:

```bash
/usr/bin/pt-heartbeat \
  --database=percona --table=heartbeat \
  --sentinel=/tmp/pt-heartbeat-sentinel \
  --log=/var/log/mysql/pt-heartbeat.log \
  --interval=0.01 --update
```


On replicas, it runs with --monitor.

We defined the log path explicitly using the --log flag and rotated logs using /etc/logrotate.d/mysql, which included a wildcard:

```bash
/var/log/mysql/*.log /var/log/mysql/*.err {
    ...
    compress
    size 100M
    rotate 5
    ...
}
```

## The Issue: Disk Space Bloat Due to Orphaned File Descriptors

Over time, we started getting disk space alerts on the root volume (/) of some DB nodes, even though:

/var/log/mysql/pt-heartbeat.log showed 0 bytes, and the .gz rotated files were regularly compressed.

```bash
root@prod-mysql-01:/home/pakumar# ls -lah /var/log/mysql/pt-heartbea*

-rw-r--r-- 1 root root 0 Aug 20 00:00 /var/log/mysql/pt-heartbeat.log 
-rw-r--r-- 1 root root 39M Aug 19 23:59 /var/log/mysql/pt-heartbeat.log.1.gz 
-rw-r--r-- 1 root root 56M Jul 4 00:00 /var/log/mysql/pt-heartbeat.log.2.gz 
-rw-r--r-- 1 root root 49K May 22 09:26 /var/log/mysql/pt-heartbeat.log.3.gz 
-rw-r--r-- 1 root root 29M May 20 00:00 /var/log/mysql/pt-heartbeat.log.4.gz 
-rw-r--r-- 1 root root 1.8M Jul 31 2024 /var/log/mysql/pt-heartbeat.log.5.gz
```


Despite this, a df -h / showed >90% usage, and we noticed space was only freed when we manually restarted the pt-heartbeat service:

```bash
systemctl restart pt-heartbeat.service
```

## Root Cause

When logrotate rotates pt-heartbeat.log, it:

- Renames the current log file to .log.1

- Creates a new, empty .log

- Compresses old logs as .gz

However, pt-heartbeat doesn’t reopen the log file automatically. It keeps writing to the old file descriptor — which is no longer linked to a filename on disk.

This results in a classic orphaned inode situation:

- File is "deleted" (renamed and compressed)

- But space is still consumed until the writing process (pt-heartbeat) is restarted

## The Fix: Dedicated Logrotate + Systemd Restart

To solve this cleanly and permanently, we made two key changes:

1. Created a separate logrotate config for pt-heartbeat

We moved its log handling out of the MySQL block:

```bash
/etc/logrotate.d/pt-heartbeat
bash

```bash
/var/log/mysql/pt-heartbeat.log {
    notifempty
    size 100M
    rotate 7
    compress
    missingok
    copytruncate
    sharedscripts
    postrotate
        if systemctl is-active --quiet pt-heartbeat-primary.service; then
            systemctl restart pt-heartbeat-primary.service
        fi
        if systemctl is-active --quiet pt-heartbeat-replica.service; then
            systemctl restart pt-heartbeat-replica.service
        fi
    endscript
}
```

2. Excluded pt-heartbeat.log from the MySQL logrotate config

We updated /etc/logrotate.d/mysql:

```bash
/var/log/mysql/*.log
! /var/log/mysql/pt-heartbeat.log
/var/log/mysql/*.err
```

This prevents accidental double-rotation or compression of the same file.

## Outcome

After implementing the fix:

- Disk usage immediately dropped from 91% to 34%

- pt-heartbeat.log is now rotated cleanly

- Old log file descriptors are released after every rotation

- No more manual restarts or space alerts

## Takeaways

- Long-running services like pt-heartbeat won’t reopen logs unless restarted.

- Logrotate must coordinate with service restarts, especially for tools that don’t support SIGHUP-based log reopening.

- Exclude such logs from broader rotation scripts (like mysql.log) to prevent side effects.
---
title: MySQL to RDS Replication Script using XtraBackup and S3
date: 2025-06-04
tags: [mysql, rds, xtrabackup, disaster-recovery, aws, s3]
---

Setting up a disaster recovery (DR) pipeline between your on-premise MySQL database and Amazon RDS can be a game-changer for data resilience. In this blog, I’ll walk through how I automated a reliable backup + replication process using Percona XtraBackup, AWS S3, and RDS’s restore-from-S3 capability.

---

## Goal

Build a script to:

- Take a full backup of MySQL using `xtrabackup`
- Split and upload it to Amazon S3 in parallel
- Restore the backup to a fresh RDS instance
- Set up GTID-based replication to start syncing with the on-prem master

---

## Pre-requisites

1. **AWS CLI Setup**
    ```bash
    aws configure
    ```

2. **IAM Role for RDS Ingestion**
    - Create a role with `AmazonS3ReadOnlyAccess`
    - Add this trust policy:
    ```json
    {
      "Effect": "Allow",
      "Principal": { "Service": "rds.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
    ```

3. **MySQL Backup User**
    ```sql
    CREATE USER 'xtrabackup_user'@'%' IDENTIFIED BY '*****';
    GRANT SELECT, RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT, REPLICATION SLAVE, BACKUP_ADMIN ON *.* TO 'xtrabackup_user'@'%';
    ```

4. **GTID Mode Enabled**
    Make sure both RDS and your MySQL source have:
    ```
    gtid_mode = ON
    enforce_gtid_consistency = ON
    log_slave_updates = ON
    ```

---

## The Script

```
#!/bin/bash
set -euo pipefail

MYSQL_USER="xtrabackup_user"
MYSQL_PASSWORD="****"

S3_BUCKET="your-s3-bucket"
AWS_REGION="us-west-2"
BACKUP_TIMESTAMP="$(date +%F_%H-%M-%S)"
S3_PREFIX="dr-backup"

LOG_DIR="/mnt/PERCONA-BACKUPS/dr_backup"
LOG_FILE="$LOG_DIR/xtrabackup_stream_${BACKUP_TIMESTAMP}.log"
THREADS=12

RDS_INSTANCE_IDENTIFIER="dr-mysql-replica"
RDS_MASTER_USER="root"
RDS_MASTER_PASSWORD="*****"
RDS_ENGINE="mysql"
RDS_ENGINE_VERSION="8.0.41"
RDS_DB_CLASS="db.t4g.large"
RDS_ALLOCATED_STORAGE=200
RDS_INGESTION_ROLE_ARN="arn:aws:iam::<account>:role/<role-name>"
VPC_SECURITY_GROUP_IDS=("sg-123" "sg-456")
DB_SUBNET_GROUP_NAME="rds-subnet"
OPTION_GROUP_NAME="default:mysql-8-0"
DB_PARAMETER_GROUP_NAME="your-param-group"

ONPREM_HOST="10.0.0.10"
ONPREM_PORT=3306
REPL_USER="repl"
REPL_PASSWORD="*****"

exec >> "$LOG_FILE" 2>&1

echo "[1] Taking backup and splitting"
xtrabackup --backup --user="$MYSQL_USER" --password="$MYSQL_PASSWORD" --stream=xbstream \
  --parallel=$THREADS --target-dir=$LOG_DIR | \
  split -d --bytes=10240MB - $LOG_DIR/backup.xbstream

echo "[2] Uploading parts in parallel"
find $LOG_DIR -type f -name 'backup.xbstream*' | \
  xargs -n 1 -P 8 -I {} bash -c 'aws s3 cp "{}" "s3://'"$S3_BUCKET"'/'"$S3_PREFIX"'/" --sse AES256 --region '"$AWS_REGION"

echo "[3] Triggering RDS restore"
aws rds restore-db-instance-from-s3 \
  --db-instance-identifier "$RDS_INSTANCE_IDENTIFIER" \
  --allocated-storage "$RDS_ALLOCATED_STORAGE" \
  --db-instance-class "$RDS_DB_CLASS" \
  --engine "$RDS_ENGINE" \
  --engine-version "$RDS_ENGINE_VERSION" \
  --master-username "$RDS_MASTER_USER" \
  --master-user-password "$RDS_MASTER_PASSWORD" \
  --s3-bucket-name "$S3_BUCKET" \
  --s3-ingestion-role-arn "$RDS_INGESTION_ROLE_ARN" \
  --s3-prefix "$S3_PREFIX" \
  --source-engine "$RDS_ENGINE" \
  --source-engine-version "$RDS_ENGINE_VERSION" \
  --region "$AWS_REGION" \
  --db-subnet-group-name "$DB_SUBNET_GROUP_NAME" \
  --vpc-security-group-ids "${VPC_SECURITY_GROUP_IDS[@]}" \
  --option-group-name "$OPTION_GROUP_NAME" \
  --db-parameter-group-name "$DB_PARAMETER_GROUP_NAME"

echo "[4] Waiting for RDS to become available..."
while true; do
  STATUS=$(aws rds describe-db-instances \
    --db-instance-identifier "$RDS_INSTANCE_IDENTIFIER" \
    --region "$AWS_REGION" \
    --query "DBInstances[0].DBInstanceStatus" \
    --output text 2>/dev/null || echo "not-found")
  echo "  ➤ $STATUS"
  case "$STATUS" in
    "available") break ;; 
    "failed"|"incompatible-restore"|"not-found") exit 1 ;; 
    *) sleep 30 ;;
  esac
done

RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier "$RDS_INSTANCE_IDENTIFIER" \
  --region "$AWS_REGION" \
  --query "DBInstances[0].Endpoint.Address" \
  --output text)

echo "[5] Setting up replication"
mysql -h "$RDS_ENDPOINT" -u "$RDS_MASTER_USER" -p"$RDS_MASTER_PASSWORD" <<EOF
CALL mysql.rds_set_external_master_with_auto_position('${ONPREM_HOST}', ${ONPREM_PORT}, '${REPL_USER}', '${REPL_PASSWORD}', 0, 0);
CALL mysql.rds_start_replication();
EOF
```

## Key Considerations

- The backup must not be compressed (--compress is not supported by RDS restore)

- Use --slave-info to capture GTID state

- Your RDS parameter group must have gtid_mode=ON

- Use split with --bytes=10240MB to avoid multipart size limits on S3

- Uploading via xargs -P 8 ensures efficient concurrency

## Summary

This setup helped us streamline disaster recovery for MySQL into Amazon RDS. The script is production-ready and can be scheduled via cron or integrated into a backup orchestration workflow.
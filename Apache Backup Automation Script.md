---

# Apache Backup Automation Script

This guide sets up a scheduled Apache server backup to a remote backup server with logging via SQLite and passwordless SSH.

---

## Step 1: Create the Backup Script

Create the shell script at:

```bash
/usr/local/bin/apache_backup.sh
```

## Step 2: Add the Script Content

Paste the following into the file:

```bash
#!/bin/bash

# Variables
SRC_DIR="/etc/httpd /var/www"
DEST_DIR="/backup_svr/Bkash_Application_Backup_2025/SERVER_100_148"
REMOTE_USER="backup"
REMOTE_HOST="192.168.20.26"
DATE=$(date +"%Y%m%d")
BACKUP_NAME="apache_backup_${DATE}.tar.gz"
TMP_DIR="/tmp/apache_backup_${DATE}"
DB_FILE="/var/log/backup_logs.db"

# Start time
START_TIME=$(date +"%Y-%m-%d %H:%M:%S")

# Create temporary directory
mkdir -p $TMP_DIR
cp -r $SRC_DIR $TMP_DIR/ 2>/dev/null

# Create tar.gz archive
tar -czf /tmp/$BACKUP_NAME -C /tmp apache_backup_${DATE}
STATUS=$?

# Upload to remote server
if [ $STATUS -eq 0 ]; then
    scp -q /tmp/$BACKUP_NAME ${REMOTE_USER}@${REMOTE_HOST}:${DEST_DIR}/
    STATUS=$?
fi

# End time
END_TIME=$(date +"%Y-%m-%d %H:%M:%S")

# Cleanup local temp files
rm -rf $TMP_DIR /tmp/$BACKUP_NAME

# Rotate old backups (keep last 7 only) and capture deleted files
DELETED_FILES=$(ssh ${REMOTE_USER}@${REMOTE_HOST} "cd ${DEST_DIR} && ls -1t apache_backup_*.tar.gz | tail -n +8")
if [ -n "$DELETED_FILES" ]; then
    ssh ${REMOTE_USER}@${REMOTE_HOST} "cd ${DEST_DIR} && echo \"$DELETED_FILES\" | xargs -r rm -f"
    REMARK=$(echo "$DELETED_FILES" | tr '\n' ',' | sed 's/,$//')
else
    REMARK=""
fi

# Backup size (on remote server after upload)
BACKUP_SIZE=$(ssh ${REMOTE_USER}@${REMOTE_HOST} "du -sh ${DEST_DIR}/${BACKUP_NAME} | cut -f1" 2>/dev/null)

# Status & Message
if [ $STATUS -eq 0 ]; then
    RESULT="success"
    MSG="Backup completed successfully"
else
    RESULT="failure"
    MSG="Backup failed"
fi

# Insert log into SQLite DB
sqlite3 $DB_FILE <<EOF
INSERT INTO apache_backup_log (starttime, endtime, source_dir, destination_dir, backupsize, status, msg, remark)
VALUES (
    '$START_TIME',
    '$END_TIME',
    '$SRC_DIR',
    '${DEST_DIR}/${BACKUP_NAME}',
    '$BACKUP_SIZE',
    '$RESULT',
    '$MSG',
    '$REMARK'
);
EOF
```

---

## Step 3: Make Script Executable

```bash
chmod +x /usr/local/bin/apache_backup.sh
```

---

## Step 4: Setup SSH Access to Backup Server

### 1. Generate SSH Key (on Apache/source server)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```

### 2. Copy Key to Backup Server

```bash
ssh-copy-id backup@192.168.20.26
```

### 3. Test SSH Login

```bash
ssh backup@192.168.20.26
```

---

## Step 5: Setup SQLite Logging

### 1. Install SQLite

```bash
sudo dnf install -y sqlite sqlite-devel
```

### 2. Create Backup Log Database

Run once:

```bash
sudo sqlite3 /var/log/backup_logs.db "CREATE TABLE IF NOT EXISTS apache_backup_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    starttime TEXT,
    endtime TEXT,
    source_dir TEXT,
    destination_dir TEXT,
    backupsize TEXT,
    status TEXT,
    msg TEXT,
    remark TEXT
);"
```

---

## Step 6: Schedule Daily Cron Job

### 1. Edit Crontab

```bash
sudo crontab -e
```

### 2. Add the following line (runs daily at 3:00 AM)

```cron
0 3 * * * /usr/local/bin/apache_backup.sh
```

---

## Notes

* **Ensure `/backup_svr/...` is accessible from the remote server.**
* **Ensure sufficient disk space on both source and destination.**
* **Verify script works manually before scheduling with cron.**
* **Review SQLite logs for any anomalies or failures.**

---

Let me know if you'd like this as a downloadable `.md` file or need help creating a GitHub repository structure (e.g., `scripts/`, `logs/`, `README.md`, etc.).

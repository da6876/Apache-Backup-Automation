# Apache Backup Automation

This project provides a **Bash script** to automate the backup of Apache configuration files and website data.  
The script compresses files, uploads them to a remote backup server, manages backup retention, and logs results in an SQLite database for tracking.

---

## 📑 Table of Contents
- [Introduction](#introduction)  
- [Features](#features)  
- [Requirements](#requirements)  
- [Installation](#installation)  
- [Configuration](#configuration)  
- [Usage](#usage)  
- [Database Schema](#database-schema)  
- [Troubleshooting](#troubleshooting)  
- [Contributors](#contributors)  
- [License](#license)  

---

## 🔹 Introduction
The **`apache_backup.sh`** script automates:
- Creating compressed backups of Apache configs (`/etc/httpd`) and websites (`/var/www`).
- Uploading archives securely to a remote server using **SSH**.
- Rotating old backups (keeping only the latest 7).
- Logging backup details in **SQLite** for audit and monitoring.

---

## ✨ Features
- Passwordless SSH backups using key authentication.  
- Automatic cleanup of old backups.  
- Lightweight logging with **SQLite3**.  
- Configurable source/destination directories.  
- Tracks size, status, timestamps, and deleted backup files.  

---

## 📦 Requirements
- **Source (Apache) server**  
  - Bash, tar, scp  
  - SQLite3  

- **Backup server**  
  - SSH access  
  - Sufficient storage  

- **Tested on**: RHEL / CentOS / Fedora  

---

## ⚙️ Installation

### 1. Create the Backup Script
Save the script at `/usr/local/bin/apache_backup.sh`:
```bash
sudo nano /usr/local/bin/apache_backup.sh
# paste the script content
```

### 2. Make the Script Executable
```bash
sudo chmod +x /usr/local/bin/apache_backup.sh
```

### 3. Set Up SSH Keys
On the Apache server:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
ssh-copy-id backup@192.168.20.26
ssh backup@192.168.20.26  # test passwordless login
```

### 4. Install SQLite and Create Log Database
```bash
sudo dnf install -y sqlite sqlite-devel
sudo sqlite3 /var/log/backup_logs.db "
CREATE TABLE IF NOT EXISTS apache_backup_log (
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

## 🔧 Configuration
Inside the script, you can adjust variables:

```bash
SRC_DIR="/etc/httpd /var/www"
DEST_DIR="/backup_svr/Bkash_Application_Backup_2025/SERVER_100_148"
REMOTE_USER="backup"
REMOTE_HOST="192.168.20.26"
DB_FILE="/var/log/backup_logs.db"
```

- `SRC_DIR`: Directories to back up.  
- `DEST_DIR`: Destination folder on the remote server.  
- `REMOTE_USER`: SSH user on the backup server.  
- `REMOTE_HOST`: Backup server IP/hostname.  
- `DB_FILE`: SQLite database file for logging.  

---

## ▶️ Usage
Run the script manually:
```bash
/usr/local/bin/apache_backup.sh
```

Or schedule daily with **cron**:
```bash
crontab -e
```
Example entry (runs daily at 2 AM):
```
0 2 * * * /usr/local/bin/apache_backup.sh
```

---

## 🗄 Database Schema
Backups are logged into `/var/log/backup_logs.db` under the table `apache_backup_log`.

**Columns:**
- `id` – Auto-increment log ID  
- `starttime` – Backup start timestamp  
- `endtime` – Backup end timestamp  
- `source_dir` – Backed-up directories  
- `destination_dir` – Remote storage path  
- `backupsize` – Size of the archive  
- `status` – `success` or `failure`  
- `msg` – Summary message  
- `remark` – List of old backups removed  

---

## 🚑 Troubleshooting
- **Permission denied (scp/ssh):**  
  Ensure SSH keys are correctly installed and permissions are set.  

- **SQLite errors:**  
  Confirm `/var/log/backup_logs.db` exists and is writable by the script.  

- **Backup not rotating:**  
  Check that `ls -1t` works on the remote backup directory.  

---

## 👥 Contributors
- Initial script & documentation: **[Your Name/Team]**

---

## 📜 License
This project is licensed under the MIT License.  
You are free to modify and distribute with attribution.  

# RMAN-automation
# Oracle RMAN Backup and Disaster Recovery Automation

*By David Lionel Muya Kafunda*


# Overview
In today's enterprise environments, robust backup and disaster recovery (DR) strategies are critical. I recently implemented an end-to-end solution using Oracle Recovery Manager (RMAN) and Bash scripts to automate the backup process, prevent overlapping jobs, manage old backups, and integrate a disaster recovery trigger. This repository provides an automated Oracle RMAN backup and disaster recovery solution using backup scripts, rsync synchronization, and restore automation. It ensures that production backups are efficiently transferred and restored to a DR server.

This solution, addresses challenges such as:
Overlapping Backups: Preventing multiple instances from running concurrently.
Control File Management: Ensuring that the correct (primary) control file is backed up.
Storage Optimization: Compressing and cleaning up obsolete backup files.
Automated DR Integration: Transferring backups to a DR server and triggering remote restores automatically.
In this article, I’ll detail the scripts, performance tuning techniques, and automation strategies used to achieve this solution. I invite you to review, provide feedback, or contribute improvements.

# Repository Structure
The solution comprises SIX scripts:

rman_backup_level0.sh: Handles Level 0 (full) backups.
rman_backup_level1.sh: Handles Level 1 (incremental) backups.
rman_archivelog.sh	Archive log backup & cleanup
rsync_backup.sh	Sync backups to DR
rsync_fra.sh	Sync archived logs
trigger_restore.sh: Triggers the restore process on the DR server.
All scripts include robust logging, error handling, and use a lock file mechanism to prevent overlapping executions.

# rman-automation
│── scripts/

│   ├── rman_backup_level0.sh

│   ├── rman_backup_level1.sh

│   ├── rman_archivelog.sh

│   ├── rsync_backup.sh

│   ├── rsync_fra.sh

│   ├── trigger_restore.sh

│── docs/

│   ├── setup_guide.md

│   ├── monitoring_logs.md

│   ├── troubleshooting.md

│── README.md

│── .gitignore

│── LICENSE


# Key Features and Techniques
1. Lock File Mechanism
To prevent multiple backup processes from running simultaneously, we use flock in combination with a lock file. Here’s an excerpt from our script:

bash
#!/bin/bash
LOCK_FILE=/tmp/rman_backup_level1.lock
exec 200>"$LOCK_FILE"
flock -n 200 || { echo "Another instance is already running. Exiting..."; exit 1; }
trap 'echo "Cleaning up lock file and exiting..."; exec 200>&-; rm -f "$LOCK_FILE"' EXIT
exec 200>"$LOCK_FILE": Opens file descriptor 200 for the lock file.
flock -n 200 prevents concurrent script executions.
trap ... EXIT ensures the lock file is removed on exit, even if the script is interrupted.
# 2. RMAN Backup Commands
Our scripts use RMAN to perform both full (Level 0) and incremental (Level 1) backups, including archivelogs and the current control file. For example, the command in our Level 0 backup script is:

rman
RUN {
    BACKUP INCREMENTAL LEVEL 0 AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
    BACKUP CURRENT CONTROLFILE FORMAT '/archive/backups/controlfile.ctl';
}
EXIT;
This ensures that the entire database, along with the archived logs and the control file, is backed up in one cohesive operation.

# 3. Automated Cleanup and Compression
To optimize disk space, our scripts automatically compress and delete backup files older than a defined retention period (e.g., 7 days). We use the find command for this purpose:

bash
echo "Zipping backups older than 7 days..."
find "$BACKUP_DIR" -type f -mtime +7 ! -name "*.gz" -exec gzip {} \;
# 4. Obsolete Backup Management with RMAN
We leverage RMAN’s retention policy to delete backups that are no longer necessary. For example:

rman
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 3 DAYS;
DELETE NOPROMPT OBSOLETE;
This configuration keeps backups required to recover the database within the last 3 days while marking older backups as obsolete.

# 5. Secure Transfer and DR Integration
After the backup completes, we use rsync to transfer backup files to our DR server:

bash
rsync -avz --ignore-existing "$BACKUP_DIR/" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR"
Finally, the script triggers the restore process on the DR server remotely via SSH:

bash
ssh "$REMOTE_USER@$REMOTE_HOST" "bash $RESTORE_TRIGGER_SCRIPT"

# Full Script Example
Below is a complete excerpt from our Level 1 backup script, which incorporates the features discussed:

bash
#!/bin/bash

# Variables
LOCK_FILE=/tmp/rman_backup_level1.lock
ORACLE_SID=[SID]
BACKUP_DIR=/archive/backups
LOG_DIR=/archive/logs
LOG_FILE=$LOG_DIR/rman_backup_level1_$(date +%Y%m%d_%H%M%S).log
REMOTE_USER=oracle
REMOTE_HOST=[IP]
REMOTE_DIR=/archive/backups
ORACLE_HOME=/d01/app/oracle/product/11.2.0/dbhome_1
PATH=$ORACLE_HOME/bin:$PATH
INCREMENTAL_LEVEL=${1:-0}  # Default to Level 0 if not passed
ZIP_DIR=/archive/zipped_backups
RESTORE_TRIGGER_SCRIPT=/home/oracle/scripts/trigger_restore.sh

# Function for logging
log() { echo "$(date +'%Y-%m-%d %H:%M:%S') - $1"; }

# Set Oracle environment
export ORACLE_SID ORACLE_HOME PATH

# Create necessary directories
mkdir -p "$BACKUP_DIR" "$ZIP_DIR" "$LOG_DIR"

# Start logging
exec > >(tee -a "$LOG_FILE") 2>&1

# Lock the script to prevent overlap
exec 200>"$LOCK_FILE"
flock -n 200 || { log "Another instance is already running. Exiting..."; exit 1; }
trap 'log "Cleaning up lock file and exiting..."; exec 200>&-; rm -f "$LOCK_FILE"' EXIT

log "Starting RMAN backup at $(date)"

# Remove existing control file backup if present
if [ -f "$BACKUP_DIR/controlfile.ctl" ]; then
    log "Deleting existing control file backup: $BACKUP_DIR/controlfile.ctl"
    rm -f "$BACKUP_DIR/controlfile.ctl"
    if [ $? -eq 0 ]; then
        log "Existing control file backup deleted successfully."
    else
        log "Failed to delete existing control file backup. Exiting..."
        exit 1
    fi
fi

# Perform RMAN backup
log "Performing level $INCREMENTAL_LEVEL backup..."
rman target / <<EOF
RUN {
    BACKUP INCREMENTAL LEVEL $INCREMENTAL_LEVEL AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
    BACKUP CURRENT CONTROLFILE FORMAT '$BACKUP_DIR/controlfile.ctl';
}
EXIT;
EOF

if [ $? -eq 0 ]; then
    log "Backup completed successfully."

    # Zip older backups (older than 7 days)
    log "Zipping backups older than 7 days..."
    find "$BACKUP_DIR" -type f -mtime +7 ! -name "*.gz" -exec gzip {} \;
    
    # Delete obsolete backups using RMAN
    log "Deleting obsolete backups..."
    rman target / <<EOF
RUN {
    CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 3 DAYS;
    DELETE NOPROMPT OBSOLETE;
}
EXIT;
EOF

    if [ $? -eq 0 ]; then
        log "Obsolete backups deleted successfully."
    else
        log "Failed to delete obsolete backups."
    fi

    # Transfer backups to DR Server
    log "Starting file transfer to $REMOTE_HOST..."
    rsync -avz --ignore-existing "$BACKUP_DIR/" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR"
    if [ $? -eq 0 ]; then
        log "File transfer to $REMOTE_HOST completed successfully."
    else
        log "File transfer failed."
    fi
else
    log "Backup failed. Exiting..."
    exit 1
fi

# Trigger the restore process on DR Server
log "Triggering restore process on DR Server..."
ssh "$REMOTE_USER@$REMOTE_HOST" "bash $RESTORE_TRIGGER_SCRIPT"
if [ $? -eq 0 ]; then
    log "Restore process triggered successfully on DR Server."
else
    log "Failed to trigger restore process on DR Server. Exiting..."
    exit 1
fi

log "Backup and transfer process to DR Server completed at $(date)"
exit 0

# Installation & Setup
# 1️⃣ Clone the Repository
bash
git clone https://github.com/yourusername/RMAN-automation.git

cd RMAN-automation

# 2️⃣ Configure Oracle Environment
Modify the environment variables in the scripts (scripts/rman_backup_level0.sh, etc.).

Update ORACLE_SID
Set REMOTE_HOST (DR Server IP)
Update BACKUP_DIR path


# Scheduling via Cron Jobs
bash

1.* * * * * /home/oracle/scripts/rsync_FRA.sh

2.*/30 * * * * /home/oracle/scripts/rman_archivelog.sh

0 * * * * /home/oracle/scripts/rsync_backup.sh

0 2 * * 0 /home/oracle/scripts/rman_backup_level0.sh

0 */4 * * 1-6 /home/oracle/scripts/rman_backup_level1.sh

# Monitoring & Logs
Check Last Backup:	tail -f /archive/logs/rman_backup_*.log
Check Lock File:	ls -l /tmp/rman_backup_level1.lock
List Backup Files:	ls -lh /archive/backups
List RMAN Backups:	rman target / LIST BACKUP SUMMARY;


# Troubleshooting
# 🛠 Backup Failed Due to Lock File
bash
rm -f /tmp/rman_backup_level1.lock
# 🛠 Backup Not Syncing to DR
bash
rsync -avzPu /archive/backups/ oracle@DR_SERVER:/archive/backups
#🛠 Restore Not Triggering
bash
ssh oracle@DR_SERVER "bash /home/oracle/scripts/trigger_restore.sh"

# Lessons Learned and Future Improvements
-Control File Management:
Ensuring that the control file is correctly backed up in primary mode (and not inadvertently as a standby) is crucial.
-Locking Mechanism:
Using flock combined with a cleanup trap prevents overlapping backup operations, ensuring data consistency.
-Automated Cleanup:
Automating compression and deletion of old backups saves storage and reduces manual intervention.
-DR Integration:
Secure and automated file transfers, coupled with a trigger for DR restores, significantly reduce downtime in the event of a failure.
Monitoring and Logging:
Detailed logging provides insights for troubleshooting and helps in maintaining a reliable backup system.


# Conclusion
This automated RMAN backup and DR solution has transformed our Oracle database management process. By leveraging RMAN, Bash scripting, and secure file transfer techniques, we now have a reliable, efficient, and automated backup framework. I invite fellow Oracle professionals and developers to share their experiences, improvements, and any further questions on this topic.

# Contributing
Fork this repository
Create a branch (git checkout -b feature-branch)
Submit a PR

Feel free to explore the repository, suggest enhancements, or open an issue if you encounter any challenges. Happy automating!

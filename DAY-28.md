# Day 28 💼 — Backup & Recovery

> **"Data backup nahi kiya toh ek din zaroor pachtaaoge!"**

---

## 🤔 Backup Kyun Zaroori Hai?

- Hard disk kabhi bhi kharab ho sakti hai
- Galti se file delete ho sakti hai
- Virus/ransomware attack
- Server crash
- Human error

**Rule: 3-2-1 Backup**
- **3** copies of data
- **2** different storage types (disk + USB)
- **1** offsite (cloud ya doosri jagah)

---

## 📁 cp — Simple Backup

```bash
# Simple copy backup
cp -r /home/rahul/Documents /backup/Documents_$(date +%Y%m%d)

# Sab home folder backup
cp -r /home/rahul/ /backup/home_backup/

# Preserve permissions aur timestamps
cp -rp source/ destination/
```

---

## 🔄 rsync — Smart Backup (Best!)

rsync sirf **changed files** copy karta hai — fast aur efficient!

```bash
# Basic sync
rsync -av source/ destination/

# Options samjho:
# -a = archive (permissions, timestamps preserve karo)
# -v = verbose (kya copy ho raha dikhao)
# -z = compress karo (network transfer ke liye)
# -h = human readable
# --delete = extra files destination se delete karo
# --dry-run = actually copy mat karo, sirf dikhao

# Full backup
rsync -avh /home/rahul/ /backup/rahul/

# Network backup (SSH se)
rsync -avzh /home/rahul/ user@server:/backup/rahul/

# Incremental (sirf changes)
rsync -avh --delete /home/rahul/ /backup/rahul/

# Dry run pehle!
rsync -avhn --delete /source/ /dest/
```

---

## 📦 tar — Archive Backup

```bash
# Dated backup banao
BACKUP_DIR="/backup"
DATE=$(date +%Y-%m-%d_%H-%M)

tar -czvf "$BACKUP_DIR/home_$DATE.tar.gz" /home/rahul/
tar -czvf "$BACKUP_DIR/etc_$DATE.tar.gz" /etc/

# Verify backup
tar -tzvf /backup/home_2024-01-15.tar.gz | head -20

# Restore karo
tar -xzvf /backup/home_2024-01-15.tar.gz -C /restore/
```

---

## ⏰ Automated Backup Script

```bash
nano /home/rahul/backup.sh
```

```bash
#!/bin/bash
# ==============================
# Automated Backup Script
# ==============================

BACKUP_DIR="/backup"
SOURCE_DIR="/home/rahul"
DATE=$(date +%Y-%m-%d)
LOG="/var/log/backup.log"
KEEP_DAYS=7       # Kitne din purana backup rakhna hai

echo "=== Backup Start: $(date) ===" >> $LOG

# Backup directory banao
mkdir -p "$BACKUP_DIR/$DATE"

# rsync se backup lo
rsync -av --delete "$SOURCE_DIR/" "$BACKUP_DIR/$DATE/" >> $LOG 2>&1

if [ $? -eq 0 ]; then
    echo "SUCCESS: Backup complete - $DATE" >> $LOG
else
    echo "ERROR: Backup failed - $DATE" >> $LOG
fi

# Purane backups delete karo
find "$BACKUP_DIR" -maxdepth 1 -type d -mtime +$KEEP_DAYS -exec rm -rf {} \;

echo "=== Backup End: $(date) ===" >> $LOG
echo "---" >> $LOG
```

```bash
chmod +x /home/rahul/backup.sh

# Cron mein add karo — roz raat 2 baje
crontab -e
# 0 2 * * * /home/rahul/backup.sh
```

---

## ☁️ Cloud Backup

```bash
# AWS S3 se backup (aws cli install karo)
aws s3 sync /home/rahul/ s3://my-bucket/rahul-backup/

# Google Drive (rclone se)
sudo apt install rclone
rclone config                     # Setup karo
rclone sync /home/rahul/ gdrive:backup/rahul/

# Dropbox (dropbox-uploader script se)
```

---

## 🔧 System Restore

```bash
# File restore karo rsync backup se
rsync -av /backup/2024-01-15/Documents/ /home/rahul/Documents/

# tar backup se restore karo
tar -xzvf /backup/home_2024-01-15.tar.gz -C /

# Specific file restore
tar -xzvf backup.tar.gz home/rahul/Documents/important.txt
```

---

## 🏋️ Practice

```bash
# Step 1: Backup folder banao
mkdir -p /tmp/mybackup

# Step 2: Test backup lo
rsync -avh ~/Documents/ /tmp/mybackup/Documents/

# Step 3: Check karo
ls -la /tmp/mybackup/Documents/

# Step 4: Script banao aur test karo
nano ~/backup_test.sh
# (upar wala script copy karo)
chmod +x ~/backup_test.sh
~/backup_test.sh

# Step 5: Log dekho
cat /var/log/backup.log
```

---

## 📝 Aaj Ka Summary

✅ 3-2-1 Rule yaad rakho
✅ `rsync` — best backup tool (incremental)
✅ `tar` — archive backup
✅ Cron se automate karo
✅ Purani backups automatically clean karo
✅ Cloud backup bhi rakho (safety ke liye)

---
**Day:** 28/30 | **Topic:** Backup & Recovery | **Difficulty:** ⭐⭐⭐☆☆

# Evidence Custodian - Backup & Recovery Guide

## 📁 What to Backup

| Component | Location | Description |
|-----------|----------|-------------|
| **Database** | `/opt/evidence-custodian/data/evidence_custodian.db` | Users, payments, invoices, evidence records |
| **Uploads** | `/opt/evidence-custodian/uploads/` | Original evidence files |
| **Certificates** | `/opt/evidence-custodian/output/` | Generated PDF/HTML/DOCX certificates |
| **QR Code** | `/opt/evidence-custodian/data/qr/` | Uploaded UPI QR code |
| **Config** | `/opt/evidence-custodian/.env` | Environment configuration |
| **Admin DB** | `/opt/evidence_admin/data/evidence_custodian.db` | Same DB (shared) |
| **Admin Config** | `/opt/evidence_admin/.env` | Admin environment config |

---

## 1️⃣ Regular Backup (Daily/Weekly)

### Option A: Quick Backup Script

Create `/opt/evidence-custodian/backup.sh`:

```bash
#!/bin/bash
# Evidence Custodian Backup Script
# Run: ./backup.sh

BACKUP_DIR="/opt/backups/evidence-custodian"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="ec_backup_${TIMESTAMP}"

# Create backup directory
mkdir -p ${BACKUP_DIR}

echo "🔄 Starting backup: ${BACKUP_NAME}"

# Stop containers briefly for consistent backup
echo "⏸️  Pausing containers..."
cd /opt/evidence-custodian
docker compose -f docker-compose.production.yml pause app

# Backup database
echo "💾 Backing up database..."
cp data/evidence_custodian.db ${BACKUP_DIR}/${BACKUP_NAME}_database.db

# Resume containers
echo "▶️  Resuming containers..."
docker compose -f docker-compose.production.yml unpause app

# Backup configuration
echo "⚙️  Backing up configuration..."
cp .env ${BACKUP_DIR}/${BACKUP_NAME}_config.env

# Backup QR code (if exists)
if [ -d "data/qr" ]; then
    echo "📱 Backing up QR code..."
    cp -r data/qr ${BACKUP_DIR}/${BACKUP_NAME}_qr
fi

# Backup uploads (optional - can be large)
# Uncomment if you want to backup original files
# echo "📁 Backing up uploads..."
# tar -czf ${BACKUP_DIR}/${BACKUP_NAME}_uploads.tar.gz uploads/

# Backup certificates (optional)
# echo "📄 Backing up certificates..."
# tar -czf ${BACKUP_DIR}/${BACKUP_NAME}_output.tar.gz output/

# Create final archive
echo "📦 Creating archive..."
cd ${BACKUP_DIR}
tar -czf ${BACKUP_NAME}.tar.gz ${BACKUP_NAME}_*
rm -f ${BACKUP_NAME}_database.db ${BACKUP_NAME}_config.env
rm -rf ${BACKUP_NAME}_qr

# Cleanup old backups (keep last 7)
echo "🧹 Cleaning old backups..."
ls -t ${BACKUP_DIR}/*.tar.gz | tail -n +8 | xargs -r rm

echo "✅ Backup complete: ${BACKUP_DIR}/${BACKUP_NAME}.tar.gz"
echo "📊 Backup size: $(du -h ${BACKUP_DIR}/${BACKUP_NAME}.tar.gz | cut -f1)"
```

### Make executable and run:
```bash
chmod +x /opt/evidence-custodian/backup.sh
./backup.sh
```

### Option B: Automated Daily Backup (Cron)

```bash
# Edit crontab
crontab -e

# Add this line (runs daily at 2 AM)
0 2 * * * /opt/evidence-custodian/backup.sh >> /var/log/ec_backup.log 2>&1
```

---

## 2️⃣ Full Backup (For Migration)

### Step 1: Stop Services
```bash
# Stop main app
cd /opt/evidence-custodian
docker compose -f docker-compose.production.yml down

# Stop admin app
cd /opt/evidence_admin
docker compose down
```

### Step 2: Create Full Backup
```bash
#!/bin/bash
# Full backup for migration
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="/opt/backups/ec_full_migration_${TIMESTAMP}.tar.gz"

echo "📦 Creating full backup for migration..."

tar -czvf ${BACKUP_FILE} \
    --exclude='*/venv/*' \
    --exclude='*/__pycache__/*' \
    --exclude='*.pyc' \
    /opt/evidence-custodian/data/ \
    /opt/evidence-custodian/uploads/ \
    /opt/evidence-custodian/output/ \
    /opt/evidence-custodian/.env \
    /opt/evidence_admin/.env \
    2>/dev/null

echo "✅ Full backup created: ${BACKUP_FILE}"
echo "📊 Size: $(du -h ${BACKUP_FILE} | cut -f1)"
```

### Step 3: Transfer to New Server
```bash
# Using SCP
scp /opt/backups/ec_full_migration_*.tar.gz user@new-server:/opt/backups/

# OR using rsync (better for large files)
rsync -avz --progress /opt/backups/ec_full_migration_*.tar.gz user@new-server:/opt/backups/
```

### Step 4: Restore on New Server
```bash
# On new server
cd /opt

# Extract backup
tar -xzvf /opt/backups/ec_full_migration_*.tar.gz -C /

# Deploy application (use latest tarball)
mkdir -p /opt/evidence-custodian
cd /opt/evidence-custodian
tar -xf evidence_custodian_v10_fixed25.tar

# Copy .env file
# (already restored from backup)

# Start services
docker compose -f docker-compose.production.yml up -d --build

# Deploy admin
mkdir -p /opt/evidence_admin
cd /opt/evidence_admin
tar -xf evidence_admin_v2_fix11.tar
mv evidence_admin/* .
rmdir evidence_admin
docker compose up -d --build
```

---

## 3️⃣ Disaster Recovery

### Scenario A: Database Corruption

```bash
# Stop app
cd /opt/evidence-custodian
docker compose -f docker-compose.production.yml down

# Restore database from backup
cp /opt/backups/ec_backup_YYYYMMDD_HHMMSS/ec_backup_*_database.db \
   /opt/evidence-custodian/data/evidence_custodian.db

# Start app
docker compose -f docker-compose.production.yml up -d
```

### Scenario B: Server Crash (Complete Rebuild)

```bash
# 1. Provision new server (Ubuntu 22.04/24.04)

# 2. Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# 3. Install Nginx
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx

# 4. Create directories
sudo mkdir -p /opt/evidence-custodian/{data,uploads,output}
sudo mkdir -p /opt/evidence_admin/data
sudo mkdir -p /opt/backups

# 5. Restore from backup
cd /opt
sudo tar -xzvf /path/to/ec_full_migration_*.tar.gz -C /

# 6. Deploy application
cd /opt/evidence-custodian
tar -xf evidence_custodian_v10_fixed25.tar

# 7. Configure environment
# Edit .env with correct values (restored from backup or recreate)

# 8. Start services
docker compose -f docker-compose.production.yml up -d --build

# 9. Configure Nginx (see nginx config below)

# 10. Setup SSL
sudo certbot --nginx -d evidence.gnugroup.org

# 11. Deploy admin
cd /opt/evidence_admin
tar -xf evidence_admin_v2_fix11.tar
mv evidence_admin/* .
docker compose up -d --build
```

### Nginx Configuration (Reference)
```nginx
# /etc/nginx/sites-available/evidence-custodian
server {
    listen 80;
    server_name evidence.gnugroup.org;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name evidence.gnugroup.org;

    ssl_certificate /etc/letsencrypt/live/evidence.gnugroup.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/evidence.gnugroup.org/privkey.pem;

    client_max_body_size 500M;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
    }
}

# Admin on port 8443
server {
    listen 8443 ssl;
    server_name evidence.gnugroup.org;

    ssl_certificate /etc/letsencrypt/live/evidence.gnugroup.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/evidence.gnugroup.org/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 4️⃣ Backup to Cloud Storage

### Option A: AWS S3
```bash
# Install AWS CLI
apt install awscli
aws configure

# Backup script addition
aws s3 cp ${BACKUP_FILE} s3://your-bucket/evidence-custodian/backups/
```

### Option B: Google Cloud Storage
```bash
# Install gsutil
apt install google-cloud-sdk

# Backup script addition
gsutil cp ${BACKUP_FILE} gs://your-bucket/evidence-custodian/backups/
```

### Option C: Rsync to Remote Server
```bash
# Add to backup script
rsync -avz ${BACKUP_DIR}/*.tar.gz backup-user@backup-server:/backups/evidence-custodian/
```

---

## 5️⃣ Database-Only Backup (Quick)

```bash
# Quick database backup (can run while app is running)
sqlite3 /opt/evidence-custodian/data/evidence_custodian.db ".backup '/opt/backups/ec_db_$(date +%Y%m%d).db'"

# Verify backup
sqlite3 /opt/backups/ec_db_$(date +%Y%m%d).db "SELECT COUNT(*) FROM users;"
```

---

## 6️⃣ Backup Verification Checklist

After any restore, verify:

```bash
# 1. Check database integrity
sqlite3 /opt/evidence-custodian/data/evidence_custodian.db "PRAGMA integrity_check;"

# 2. Check tables exist
sqlite3 /opt/evidence-custodian/data/evidence_custodian.db ".tables"

# 3. Check user count
sqlite3 /opt/evidence-custodian/data/evidence_custodian.db "SELECT COUNT(*) FROM users;"

# 4. Check payment records
sqlite3 /opt/evidence-custodian/data/evidence_custodian.db "SELECT COUNT(*) FROM payment_orders;"

# 5. Check app is running
curl -s http://localhost:5000/health || echo "App not responding"

# 6. Check admin is running
curl -s http://localhost:8001/health || echo "Admin not responding"
```

---

## 📋 Backup Schedule Recommendation

| Type | Frequency | Retention | Storage |
|------|-----------|-----------|---------|
| Database only | Every 6 hours | 7 days | Local |
| Full backup | Daily | 30 days | Local + Cloud |
| Migration backup | Before changes | Permanent | Cloud |

---

## ⚠️ Important Notes

1. **Always test restores** - A backup is only useful if you can restore it
2. **Encrypt sensitive backups** - Database contains user data
3. **Store backups off-site** - Local backup won't help if server is destroyed
4. **Document your .env** - Keep a secure copy of environment variables
5. **Monitor backup jobs** - Set up alerts for failed backups

---

## 🔐 Encrypting Backups

```bash
# Encrypt backup with password
tar -czf - /opt/evidence-custodian/data/ | \
    openssl enc -aes-256-cbc -salt -pbkdf2 > backup_encrypted.tar.gz.enc

# Decrypt
openssl enc -d -aes-256-cbc -pbkdf2 -in backup_encrypted.tar.gz.enc | \
    tar -xzf - -C /opt/
```

---

## 📞 Emergency Contacts

- **Server Provider**: Linode Support
- **Domain Registrar**: [Your registrar]
- **SSL Certificate**: Let's Encrypt (auto-renews)
- **Developer**: [Your contact]

---

*Last updated: December 2024*
*Evidence Custodian v10.25*

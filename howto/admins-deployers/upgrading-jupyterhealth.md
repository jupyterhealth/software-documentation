# Upgrading JupyterHealth Exchange

This guide shows you how to upgrade JupyterHealth Exchange to a newer version safely.

### Preparation

- [Prerequisites](#prerequisites)
- [Pre-Upgrade Checklist](#pre-upgrade-checklist)
- [Backup Production Data](#backup-production-data)

### Upgrade Process

- [Stop Production Services](#stop-production-services)
- [Update Application Code](#update-application-code)
- [Run Database Migrations](#run-database-migrations)
- [Update Static Files](#update-static-files)

### Restart and Verify

- [Restart Services](#restart-services)
- [Post-Upgrade Verification](#post-upgrade-verification)

### Recovery and Special Cases

- [Rollback Procedure (If Needed)](#rollback-procedure-if-needed)
- [Upgrade Specific Components](#upgrade-specific-components)

### Best Practices

- [Best Practices](#best-practices)

### Reference

- [Related Documentation](#related-documentation)

______________________________________________________________________

## Prerequisites

- SSH access to production server
- Database backup tools configured
- Downtime maintenance window scheduled
- Git access to repository
- `pipenv` or `pip` installed

## Pre-Upgrade Checklist

### 1. Review Release Notes

Check for breaking changes, new features, and migration notes:

```bash
# View latest releases
git fetch --tags
git tag -l | tail -5

# Read release notes
git show v2.1.0:CHANGELOG.md
```

### 2. Check Compatibility

Verify dependencies are compatible with your environment:

```bash
# Check Python version requirement
grep "python_version" Pipfile

# Check Django version
grep "django" Pipfile
```

Current requirements:

- Python: 3.10, 3.11, 3.12, or 3.13
- Django: 5.2
- PostgreSQL: 13+

Reference: `jupyterhealth-exchange/README.md`

### 3. Notify Users

Send maintenance notification:

```bash
# Template email
Subject: JupyterHealth Exchange Maintenance - [DATE] [TIME]

Dear JupyterHealth Users,

We will be performing a system upgrade on [DATE] from [START TIME] to [END TIME] [TIMEZONE].

During this time:
- The Exchange will be unavailable
- Data uploads will fail
- API requests will return 503 errors

No data will be lost. All services will resume automatically after the upgrade.

Thank you for your patience.
```

## Backup Production Data

### 1. Backup Database

Create full database backup with timestamp:

```bash
# Set variables
BACKUP_DIR="/backups/jhe"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/jhe-pre-upgrade-$TIMESTAMP.sql"

# Create backup directory
sudo mkdir -p $BACKUP_DIR

# Backup database
sudo -u postgres pg_dump jhe_production > $BACKUP_FILE

# Compress backup
gzip $BACKUP_FILE

# Verify backup size
ls -lh $BACKUP_FILE.gz
```

For encrypted backups:

```bash
# Backup with encryption
sudo -u postgres pg_dump jhe_production | \
  gpg --encrypt --recipient admin@yourdomain.com \
  > $BACKUP_FILE.gpg
```

### 2. Backup Environment Configuration

```bash
# Backup .env file
sudo cp /opt/jupyterhealth-exchange/.env \
  $BACKUP_DIR/.env-pre-upgrade-$TIMESTAMP

# Backup nginx config
sudo cp /etc/nginx/sites-available/jhe \
  $BACKUP_DIR/nginx-jhe-$TIMESTAMP

# Backup systemd service
sudo cp /etc/systemd/system/jhe.service \
  $BACKUP_DIR/jhe.service-$TIMESTAMP
```

### 3. Backup Static Files (Optional)

```bash
# If using local static file storage
sudo tar -czf $BACKUP_DIR/staticfiles-$TIMESTAMP.tar.gz \
  /opt/jupyterhealth-exchange/staticfiles/
```

### 4. Test Backup Restoration

**Critical**: Verify backup can be restored before proceeding:

```bash
# Create test database
sudo -u postgres createdb jhe_test

# Restore backup
gunzip < $BACKUP_FILE.gz | sudo -u postgres psql jhe_test

# Verify data
sudo -u postgres psql jhe_test -c "SELECT COUNT(*) FROM core_observation;"
sudo -u postgres psql jhe_test -c "SELECT COUNT(*) FROM core_patient;"

# Drop test database
sudo -u postgres dropdb jhe_test
```

## Stop Production Services

### 1. Enable Maintenance Mode (Optional)

If using a maintenance page, enable it:

```bash
# Nginx maintenance mode
sudo cp /etc/nginx/maintenance.html /usr/share/nginx/html/
sudo nano /etc/nginx/sites-available/jhe
```

Add before existing location blocks:

```nginx
location / {
    return 503;
}

error_page 503 @maintenance;
location @maintenance {
    root /usr/share/nginx/html;
    rewrite ^(.*)$ /maintenance.html break;
}
```

Reload nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 2. Stop Application Server

```bash
# For systemd service
sudo systemctl stop jhe

# Verify stopped
sudo systemctl status jhe

# For docker
docker-compose down

# For supervisor
sudo supervisorctl stop jhe
```

### 3. Verify No Active Connections

```bash
# Check for active database connections
sudo -u postgres psql -c "
SELECT pid, usename, application_name, client_addr, state
FROM pg_stat_activity
WHERE datname = 'jhe_production'
AND state = 'active';
"

# Terminate connections if necessary
sudo -u postgres psql -c "
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'jhe_production'
AND pid <> pg_backend_pid();
"
```

## Update Application Code

### 1. Pull Latest Code

```bash
cd /opt/jupyterhealth-exchange

# Fetch latest changes
sudo -u jheapp git fetch --all --tags

# Check current version
git describe --tags

# Checkout new version
sudo -u jheapp git checkout v2.1.0

# Verify checkout
git describe --tags
```

### 2. Update Dependencies

```bash
# Using pipenv
sudo -u jheapp pipenv sync

# Using pip with venv
source venv/bin/activate
pip install -r requirements.txt

# Verify critical packages
pipenv run python -c "import django; print(f'Django: {django.VERSION}')"
```

### 3. Review Environment Variable Changes

```bash
# Compare .env files
diff dot_env_example.txt /opt/jupyterhealth-exchange/.env

# Check for new required variables
grep -v "^#" dot_env_example.txt | grep -v "^$" | \
  while read line; do
    var_name=$(echo $line | cut -d= -f1)
    if ! grep -q "^$var_name=" /opt/jupyterhealth-exchange/.env; then
      echo "Missing variable: $var_name"
    fi
  done
```

Add any missing variables to `.env`.

## Run Database Migrations

### 1. Review Pending Migrations

```bash
cd /opt/jupyterhealth-exchange

# Check migration status
sudo -u jheapp pipenv run python manage.py showmigrations

# List unapplied migrations
sudo -u jheapp pipenv run python manage.py showmigrations --plan | grep "\[ \]"
```

### 2. Backup Database Again (Before Migrations)

```bash
# Quick backup before migrations
sudo -u postgres pg_dump jhe_production | \
  gzip > $BACKUP_DIR/jhe-pre-migration-$TIMESTAMP.sql.gz
```

### 3. Run Migrations

```bash
# Run migrations
sudo -u jheapp pipenv run python manage.py migrate

# Check for errors
echo $?  # Should output 0
```

If migrations fail, restore backup:

```bash
# Stop application
sudo systemctl stop jhe

# Drop and recreate database
sudo -u postgres psql -c "DROP DATABASE jhe_production;"
sudo -u postgres psql -c "CREATE DATABASE jhe_production OWNER jheuser;"

# Restore backup
gunzip < $BACKUP_DIR/jhe-pre-migration-$TIMESTAMP.sql.gz | \
  sudo -u postgres psql jhe_production

# Check data integrity
sudo -u postgres psql jhe_production -c "SELECT COUNT(*) FROM core_observation;"
```

### 4. Verify Migration Success

```bash
# Check all migrations applied
sudo -u jheapp pipenv run python manage.py showmigrations | grep "\[ \]"

# Should show no unchecked boxes
```

## Update Static Files

### 1. Collect Static Files

```bash
cd /opt/jupyterhealth-exchange

# Collect static files
sudo -u jheapp pipenv run python manage.py collectstatic --no-input

# Verify files collected
ls -la staticfiles/
```

Reference: `jupyterhealth-exchange/README.md`

### 2. Update Static File Permissions

```bash
# Ensure nginx can read files
sudo chown -R jheapp:www-data staticfiles/
sudo chmod -R 755 staticfiles/
```

## Restart Services

### 1. Start Application Server

```bash
# For systemd
sudo systemctl start jhe

# Wait for startup
sleep 5

# Check status
sudo systemctl status jhe
```

### 2. Verify Application Started

```bash
# Check logs for errors
sudo journalctl -u jhe -n 50 --no-pager

# Check application responds
curl -I http://localhost:8000/admin/

# Should return HTTP 200 or 302 redirect
```

### 3. Disable Maintenance Mode

If you enabled maintenance mode, remove it:

```bash
# Remove maintenance configuration
sudo nano /etc/nginx/sites-available/jhe

# Remove the maintenance location blocks added earlier

# Test configuration
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

### 4. Verify External Access

```bash
# Test HTTPS endpoint
curl -I https://jhe.yourdomain.com/admin/

# Should return HTTP 200 or 302 redirect
```

## Post-Upgrade Verification

### 1. Smoke Test Critical Paths

#### Test Admin Login

```bash
# Navigate to admin panel
https://jhe.yourdomain.com/admin/

# Verify login works
# Check dashboard loads
```

#### Test Patient Authentication

```bash
# Generate test invitation link
curl https://jhe.yourdomain.com/api/v1/patients/10001/invitation_link \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"

# Verify link format
```

#### Test Observation Upload

```bash
# Upload test observation
curl -X POST https://jhe.yourdomain.com/fhir/r5/Observation \
  -H "Authorization: Bearer $PATIENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d @test-observation.json

# Should return 201 Created
```

Reference: `jupyterhealth-exchange/core/views/observation.py`

#### Test Observation Retrieval

```bash
# Retrieve observations
curl "https://jhe.yourdomain.com/fhir/r5/Observation?patient=10001" \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"

# Should return FHIR Bundle
```

### 2. Check Database Performance

```bash
# Check query performance
sudo -u postgres psql jhe_production -c "
SELECT schemaname, tablename, n_tup_ins, n_tup_upd, n_tup_del
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_tup_ins DESC
LIMIT 10;
"

# Check for missing indexes
sudo -u postgres psql jhe_production -c "
SELECT schemaname, tablename, attname, n_distinct, correlation
FROM pg_stats
WHERE schemaname = 'public'
AND n_distinct < 0
ORDER BY n_distinct
LIMIT 10;
"
```

### 3. Monitor Logs

```bash
# Watch logs in real-time
sudo journalctl -u jhe -f

# Check for errors in last hour
sudo journalctl -u jhe --since "1 hour ago" | grep -i error

# Check for warnings
sudo journalctl -u jhe --since "1 hour ago" | grep -i warning
```

### 4. Verify Scheduled Tasks

If using celery or cron jobs:

```bash
# Check celery workers
sudo systemctl status celery-worker

# Check cron jobs
sudo crontab -l -u jheapp

# Verify last run times
ls -lt /var/log/cron/
```

### 5. Monitor Resource Usage

```bash
# Check CPU and memory
htop

# Check disk space
df -h

# Check database size
sudo -u postgres psql jhe_production -c "
SELECT pg_size_pretty(pg_database_size('jhe_production')) AS size;
"
```

## Rollback Procedure (If Needed)

If upgrade fails and cannot be fixed quickly:

### 1. Stop Application

```bash
sudo systemctl stop jhe
```

### 2. Restore Code

```bash
cd /opt/jupyterhealth-exchange

# Checkout previous version
sudo -u jheapp git checkout v2.0.0

# Restore old dependencies
sudo -u jheapp pipenv sync
```

### 3. Restore Database

```bash
# Drop current database
sudo -u postgres psql -c "DROP DATABASE jhe_production;"

# Recreate database
sudo -u postgres psql -c "
CREATE DATABASE jhe_production
OWNER jheuser
ENCODING 'UTF8';
"

# Restore backup
gunzip < $BACKUP_DIR/jhe-pre-upgrade-$TIMESTAMP.sql.gz | \
  sudo -u postgres psql jhe_production
```

### 4. Restore Configuration

```bash
# Restore .env
sudo cp $BACKUP_DIR/.env-pre-upgrade-$TIMESTAMP \
  /opt/jupyterhealth-exchange/.env

# Restore nginx config
sudo cp $BACKUP_DIR/nginx-jhe-$TIMESTAMP \
  /etc/nginx/sites-available/jhe

sudo nginx -t
sudo systemctl reload nginx
```

### 5. Restart Services

```bash
sudo systemctl start jhe
sudo systemctl status jhe
```

### 6. Verify Rollback

```bash
# Check version
cd /opt/jupyterhealth-exchange
git describe --tags

# Test critical paths
curl -I https://jhe.yourdomain.com/admin/
```

## Upgrade Specific Components

### Upgrade Django Only

If upgrading Django minor version (e.g., 5.2.1 to 5.2.2):

```bash
# Update Pipfile
sudo nano Pipfile

# Change:
# django = "==5.2.1"
# To:
# django = "==5.2.2"

# Update dependencies
sudo -u jheapp pipenv update django

# Run migrations (usually none for minor updates)
sudo -u jheapp pipenv run python manage.py migrate

# Restart
sudo systemctl restart jhe
```

### Upgrade PostgreSQL

When upgrading PostgreSQL (e.g., 13 to 15):

```bash
# Backup data
sudo -u postgres pg_dumpall > /backups/pg_all_backup.sql

# Install new PostgreSQL version
sudo apt update
sudo apt install postgresql-15

# Stop old cluster
sudo systemctl stop postgresql@13-main

# Upgrade cluster
sudo -u postgres /usr/lib/postgresql/15/bin/pg_upgrade \
  --old-datadir=/var/lib/postgresql/13/main \
  --new-datadir=/var/lib/postgresql/15/main \
  --old-bindir=/usr/lib/postgresql/13/bin \
  --new-bindir=/usr/lib/postgresql/15/bin

# Start new cluster
sudo systemctl start postgresql@15-main

# Update connection settings in .env if needed
```

### Upgrade Python Version

When upgrading Python (e.g., 3.10 to 3.12):

```bash
# Install new Python version
sudo apt install python3.12 python3.12-venv python3.12-dev

# Recreate virtualenv
cd /opt/jupyterhealth-exchange
sudo -u jheapp python3.12 -m venv venv

# Reinstall dependencies
sudo -u jheapp venv/bin/pip install -r requirements.txt

# Test application
sudo -u jheapp venv/bin/python manage.py check

# Restart
sudo systemctl restart jhe
```

## Best Practices

### 1. Test Upgrade in Staging First

Always test upgrades in staging environment:

```bash
# Clone production to staging
pg_dump -h prod-db jhe_production | psql -h staging-db jhe_staging

# Run upgrade in staging
# Test thoroughly
# Document any issues
```

### 2. Schedule Upgrades During Low Usage

Choose upgrade window based on usage analytics:

```sql
-- Analyze usage patterns
SELECT
    date_trunc('hour', created) AS hour,
    COUNT(*) AS uploads
FROM core_observation
WHERE created > NOW() - INTERVAL '7 days'
GROUP BY hour
ORDER BY uploads DESC;
```

### 3. Keep Upgrade Documentation

Document each upgrade for future reference:

```bash
# Create upgrade log
cat >> /opt/jupyterhealth-exchange/UPGRADE_LOG.md <<EOF
## Upgrade to v2.1.0 - $(date)

**Performed by:** $(whoami)
**Start time:** $(date)
**Downtime:** 15 minutes

**Changes:**
- Upgraded Django 5.2.0 to 5.2.1
- Added new environment variable: NEW_FEATURE_FLAG
- Ran 3 migrations (core 0042, 0043, 0044)

**Issues encountered:**
- None

**Rollback plan:**
- Backup available at: $BACKUP_FILE.gz

**Verification:**
- All smoke tests passed
- No errors in logs

EOF
```

### 4. Monitor After Upgrade

Continue monitoring for 24-48 hours:

```bash
# Set up alerting for errors
watch -n 60 'sudo journalctl -u jhe --since "5 minutes ago" | grep -i error | wc -l'

# Monitor performance
watch -n 60 'curl -w "\nTime: %{time_total}s\n" -o /dev/null -s https://jhe.yourdomain.com/admin/'
```

## Related Documentation

- [Configuring JupyterHealth for HIPAA-Compliant Storage](configuring-hipaa-storage.md)
- [Troubleshooting Common Ingestion Errors](troubleshooting-ingestion-errors.md)
- [JupyterHealth Exchange Tutorial](../../tutorial/index.md)

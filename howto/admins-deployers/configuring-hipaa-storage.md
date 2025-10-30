# Configuring JupyterHealth for HIPAA-Compliant Storage

This guide shows you how to configure JupyterHealth Exchange for HIPAA-compliant patient data storage.

> **Note:** This guide includes instructions for AWS RDS, Google Cloud SQL, Azure Database for PostgreSQL, and self-hosted deployments. Before proceeding, verify command syntax and feature availability in your cloud provider's current documentation (links provided in each section).

### Initial Setup

- [Prerequisites](#prerequisites)

### Database Configuration

- [Configure Database Encryption](#configure-database-encryption)

### Application Security

- [Configure HTTPS for All Connections](#configure-https-for-all-connections)
- [Configure Secure Session Management](#configure-secure-session-management)
- [Configure OAuth2 Security](#configure-oauth2-security)

### Monitoring and Compliance

- [Configure Audit Logging](#configure-audit-logging)
- [Verify HIPAA Compliance](#verify-hipaa-compliance)

### Operations

- [Backup and Disaster Recovery](#backup-and-disaster-recovery)
- [Additional Security Measures](#additional-security-measures)

### Reference

- [Related Documentation](#related-documentation)

______________________________________________________________________

## Prerequisites

- PostgreSQL 13+ server with encryption at rest enabled
- SSL/TLS certificates for HTTPS
- Environment file (`.env`) access
- Database admin credentials
- Cloud provider CLI installed (if using cloud deployment):
  - AWS: `aws-cli` ([installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
  - Google Cloud: `gcloud` ([installation guide](https://cloud.google.com/sdk/docs/install))
  - Azure: `az` ([installation guide](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli))

## Configure Database Encryption

### 1. Enable PostgreSQL Encryption at Rest

PostgreSQL handles encryption at the storage level. Configure your PostgreSQL server for encrypted storage:

#### Before You Start

Verify current CLI syntax and features in your cloud provider's documentation:

- **AWS**: [RDS Encryption Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html)
- **Google Cloud**: [Cloud SQL Encryption (CMEK)](https://cloud.google.com/sql/docs/postgres/cmek)
- **Azure**: [Azure Database for PostgreSQL Security](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-security)

#### AWS RDS

```bash
# Enable encryption when creating the database
aws rds create-db-instance \
  --db-instance-identifier jhe-production \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 15.4 \
  --master-username jheadmin \
  --master-user-password your-secure-password \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/your-key-id \
  --backup-retention-period 7 \
  --no-publicly-accessible \
  --vpc-security-group-ids sg-xxxxxxxxx
```

Verify encryption:

```bash
aws rds describe-db-instances \
  --db-instance-identifier jhe-production \
  --query 'DBInstances[0].StorageEncrypted'
# Should return: true
```

#### Google Cloud SQL

```bash
# Create encrypted PostgreSQL instance
gcloud sql instances create jhe-production \
  --database-version=POSTGRES_15 \
  --tier=db-custom-2-7680 \
  --region=us-central1 \
  --storage-type=SSD \
  --storage-size=100GB \
  --storage-auto-increase \
  --backup \
  --retained-backups-count=7 \
  --retained-transaction-log-days=7 \
  --no-assign-ip

# Note: Encryption at rest is enabled by default in Cloud SQL
# Data is encrypted using Google-managed or customer-managed encryption keys (CMEK)

# To use customer-managed encryption keys (CMEK):
gcloud sql instances create jhe-production \
  --database-version=POSTGRES_15 \
  --tier=db-custom-2-7680 \
  --region=us-central1 \
  --disk-encryption-key=projects/PROJECT_ID/locations/LOCATION/keyRings/KEYRING/cryptoKeys/KEY
```

Create database and user:

```bash
# Create database
gcloud sql databases create jhe_production \
  --instance=jhe-production

# Create user
gcloud sql users create jheuser \
  --instance=jhe-production \
  --password=your-secure-password
```

Verify encryption:

```bash
gcloud sql instances describe jhe-production \
  --format="value(diskEncryptionConfiguration)"
```

#### Azure Database for PostgreSQL

```bash
# Create resource group (if not exists)
az group create \
  --name jhe-rg \
  --location eastus

# Create PostgreSQL server with encryption
az postgres flexible-server create \
  --resource-group jhe-rg \
  --name jhe-production \
  --location eastus \
  --admin-user jheadmin \
  --admin-password your-secure-password \
  --sku-name Standard_D2s_v3 \
  --tier GeneralPurpose \
  --storage-size 128 \
  --version 15 \
  --backup-retention 7 \
  --storage-auto-grow Enabled \
  --public-access None

# Note: Encryption at rest is enabled by default in Azure
# Data is encrypted using Microsoft-managed or customer-managed keys

# To use customer-managed keys:
az postgres flexible-server update \
  --resource-group jhe-rg \
  --name jhe-production \
  --key-name your-key-name \
  --key-vault-uri https://your-keyvault.vault.azure.net
```

Create database:

```bash
az postgres flexible-server db create \
  --resource-group jhe-rg \
  --server-name jhe-production \
  --database-name jhe_production
```

Verify encryption:

```bash
az postgres flexible-server show \
  --resource-group jhe-rg \
  --name jhe-production \
  --query "dataEncryption"
```

#### Self-Hosted PostgreSQL

Enable full disk encryption at the OS level or use PostgreSQL's pgcrypto extension for column-level encryption.

### 2. Enforce SSL Connections

#### Configure SSL in Application

In your `.env` file, configure the database to require SSL:

```bash
DB_NAME="jhe_production"
DB_USER="jheuser"
DB_PASSWORD="secure_password_here"
DB_HOST="your-db-host.com"
DB_PORT=5432
DB_SSL_MODE="require"  # Options: disable, allow, prefer, require, verify-ca, verify-full
```

For maximum security, use `verify-full` with certificate validation.

#### Configure SSL on Database Server

**AWS RDS:**

SSL/TLS is enabled by default. To enforce SSL connections:

```bash
# Create parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name jhe-pg-ssl-required \
  --db-parameter-group-family postgres15 \
  --description "Force SSL connections"

# Set rds.force_ssl parameter
aws rds modify-db-parameter-group \
  --db-parameter-group-name jhe-pg-ssl-required \
  --parameters "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate"

# Apply to instance
aws rds modify-db-instance \
  --db-instance-identifier jhe-production \
  --db-parameter-group-name jhe-pg-ssl-required \
  --apply-immediately
```

Download RDS CA certificate:

```bash
wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

Update `.env`:

```bash
DB_SSL_MODE="verify-full"
DB_SSL_ROOT_CERT="/path/to/global-bundle.pem"
```

**Google Cloud SQL:**

SSL/TLS is enabled by default. To require SSL connections:

```bash
# Require SSL
gcloud sql instances patch jhe-production \
  --require-ssl
```

Download server CA certificate:

```bash
gcloud sql ssl-certs describe server-ca \
  --instance=jhe-production \
  --format="get(cert)" > server-ca.pem
```

Create client certificate (for mutual TLS):

```bash
gcloud sql ssl-certs create client-cert \
  --instance=jhe-production \
  client-key.pem

# Download certificate
gcloud sql ssl-certs describe client-cert \
  --instance=jhe-production \
  --format="get(cert)" > client-cert.pem
```

Update `.env`:

```bash
DB_SSL_MODE="verify-full"
DB_SSL_ROOT_CERT="/path/to/server-ca.pem"
DB_SSL_CERT="/path/to/client-cert.pem"  # Optional: for mutual TLS
DB_SSL_KEY="/path/to/client-key.pem"    # Optional: for mutual TLS
```

**Azure Database for PostgreSQL:**

SSL is enabled by default and required. To download SSL certificate:

```bash
# Download certificate
wget https://dl.cacerts.digicert.com/DigiCertGlobalRootCA.crt.pem

# Or for older services
wget https://www.digicert.com/CACerts/BaltimoreCyberTrustRoot.crt.pem
```

Verify SSL is required:

```bash
az postgres flexible-server parameter show \
  --resource-group jhe-rg \
  --server-name jhe-production \
  --name require_secure_transport
# Should return: "ON"
```

Update `.env`:

```bash
DB_SSL_MODE="verify-full"
DB_SSL_ROOT_CERT="/path/to/DigiCertGlobalRootCA.crt.pem"
```

### 3. Configure Connection Pooling with Encryption

If using connection pooling (e.g., PgBouncer), ensure SSL is enabled in the pooler configuration:

```ini
# pgbouncer.ini
[databases]
jhe_production = host=your-db-host.com port=5432 dbname=jhe_production

[pgbouncer]
pool_mode = transaction
max_client_conn = 100
default_pool_size = 20
client_tls_sslmode = require
server_tls_sslmode = require
```

## Configure HTTPS for All Connections

### 1. Generate or Obtain SSL Certificates

For production, use certificates from a trusted CA (Let's Encrypt, DigiCert, etc.):

```bash
# Using certbot for Let's Encrypt
sudo certbot certonly --standalone -d jhe.yourdomain.com
```

### 2. Configure Nginx Reverse Proxy

Create nginx configuration at `/etc/nginx/sites-available/jhe`:

```nginx
server {
    listen 80;
    server_name jhe.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name jhe.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/jhe.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/jhe.yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    client_max_body_size 10M;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /opt/jupyterhealth-exchange/staticfiles/;
    }
}
```

Enable the configuration:

```bash
sudo ln -s /etc/nginx/sites-available/jhe /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 3. Update Django Settings

In your `.env` file:

```bash
SITE_URL="https://jhe.yourdomain.com"
SECURE_SSL_REDIRECT=True
SESSION_COOKIE_SECURE=True
CSRF_COOKIE_SECURE=True
```

## Configure Secure Session Management

### 1. Set Strong Secret Key

Generate a cryptographically secure secret key:

```bash
openssl rand -base64 32
```

Add to `.env`:

```bash
SECRET_KEY="your-generated-secret-key-here"
```

**Never** commit this to version control or reuse across environments.

### 2. Configure Session Settings

In `jhe/settings.py`, these security settings are already configured:

```python
SESSION_COOKIE_SECURE = True  # Only send cookies over HTTPS
SESSION_COOKIE_HTTPONLY = True  # Prevent JavaScript access
SESSION_COOKIE_SAMESITE = "Lax"  # CSRF protection
CSRF_COOKIE_SECURE = True
```

Reference: `jupyterhealth-exchange/jhe/settings.py`

## Configure OAuth2 Security

### 1. Generate RS256 Keys for OIDC

Generate a private key for signing JWTs:

```bash
openssl genrsa -out oauth2_private.pem 4096
```

Convert to JWKS format and add to Django admin:

1. Navigate to `/admin/oauth2_provider/application/`
1. Create new application with:
   - Client type: `Public`
   - Authorization grant type: `Authorization code`
   - Algorithm: `RS256`
   - Skip authorization: `Checked`
1. Paste the RSA private key in the appropriate field

Reference: See `jupyterhealth-exchange/README.md` for detailed OIDC setup

### 2. Configure PKCE for Patient Authentication

Generate static PKCE values for patient authorization flow:

```bash
# Generate code verifier (43-128 characters, base64url encoded)
CODE_VERIFIER=$(openssl rand -base64 64 | tr -d '=' | tr '+/' '-_' | cut -c1-43)

# Generate code challenge (SHA256 hash of verifier, base64url encoded)
CODE_CHALLENGE=$(echo -n "$CODE_VERIFIER" | openssl dgst -binary -sha256 | base64 | tr -d '=' | tr '+/' '-_')

echo "PATIENT_AUTHORIZATION_CODE_VERIFIER=$CODE_VERIFIER"
echo "PATIENT_AUTHORIZATION_CODE_CHALLENGE=$CODE_CHALLENGE"
```

Add to `.env`:

```bash
PATIENT_AUTHORIZATION_CODE_VERIFIER="generated-verifier-here"
PATIENT_AUTHORIZATION_CODE_CHALLENGE="generated-challenge-here"
```

Reference: `jupyterhealth-exchange/jhe/settings.py`

## Configure Audit Logging

### 1. Enable Django Logging

JupyterHealth Exchange logs are configured in `jhe/settings.py`. Ensure logs are directed to a secure, append-only location:

```bash
# In .env
DJANGO_LOG_LEVEL="INFO"
```

For production, configure log aggregation:

```python
# In jhe/settings.py, add:
LOGGING["handlers"]["file"] = {
    "level": "INFO",
    "class": "logging.handlers.RotatingFileHandler",
    "filename": "/var/log/jhe/application.log",
    "maxBytes": 1024 * 1024 * 100,  # 100MB
    "backupCount": 10,
    "formatter": "verbose",
}
```

### 2. Configure Database-Level Audit Trail

All sensitive models include `last_updated` timestamps. For comprehensive auditing, enable PostgreSQL audit logging:

#### Self-Hosted PostgreSQL

```sql
-- In postgresql.conf
log_statement = 'mod'  # Log all modifications
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

#### AWS RDS

```bash
# Create parameter group with logging enabled
aws rds create-db-parameter-group \
  --db-parameter-group-name jhe-pg-audit \
  --db-parameter-group-family postgres15 \
  --description "Audit logging enabled"

# Configure logging parameters
aws rds modify-db-parameter-group \
  --db-parameter-group-name jhe-pg-audit \
  --parameters \
    "ParameterName=log_statement,ParameterValue=mod,ApplyMethod=immediate" \
    "ParameterName=log_connections,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=log_disconnections,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=log_duration,ParameterValue=1,ApplyMethod=immediate"

# Apply to instance
aws rds modify-db-instance \
  --db-instance-identifier jhe-production \
  --db-parameter-group-name jhe-pg-audit \
  --apply-immediately

# Enable CloudWatch Logs export
aws rds modify-db-instance \
  --db-instance-identifier jhe-production \
  --cloudwatch-logs-export-configuration \
    '{"LogTypesToEnable":["postgresql"]}'
```

View logs in CloudWatch:

```bash
aws logs tail /aws/rds/instance/jhe-production/postgresql --follow
```

#### Google Cloud SQL

```bash
# Enable query logging
gcloud sql instances patch jhe-production \
  --database-flags=log_statement=mod,log_connections=on,log_disconnections=on

# View logs
gcloud logging read "resource.type=cloudsql_database AND resource.labels.database_id=PROJECT_ID:jhe-production" \
  --limit 50 \
  --format json
```

Export logs to BigQuery for long-term retention:

```bash
# Create log sink
gcloud logging sinks create jhe-audit-logs \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/jhe_logs \
  --log-filter='resource.type="cloudsql_database" AND resource.labels.database_id="PROJECT_ID:jhe-production"'
```

#### Azure Database for PostgreSQL

```bash
# Enable server logs
az postgres flexible-server parameter set \
  --resource-group jhe-rg \
  --server-name jhe-production \
  --name log_statement \
  --value MOD

az postgres flexible-server parameter set \
  --resource-group jhe-rg \
  --server-name jhe-production \
  --name log_connections \
  --value ON

az postgres flexible-server parameter set \
  --resource-group jhe-rg \
  --server-name jhe-production \
  --name log_disconnections \
  --value ON

# Configure Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group jhe-rg \
  --workspace-name jhe-logs

# Enable diagnostic settings
az monitor diagnostic-settings create \
  --resource /subscriptions/SUBSCRIPTION_ID/resourceGroups/jhe-rg/providers/Microsoft.DBforPostgreSQL/flexibleServers/jhe-production \
  --name jhe-diagnostics \
  --workspace jhe-logs \
  --logs '[{"category":"PostgreSQLLogs","enabled":true}]'
```

Query logs:

```bash
az monitor log-analytics query \
  --workspace jhe-logs \
  --analytics-query "PostgreSQLLogs | where TimeGenerated > ago(1h)" \
  --output table
```

## Verify HIPAA Compliance

### 1. Test Encryption in Transit

#### Application HTTPS

```bash
# Verify HTTPS is enforced
curl -I http://jhe.yourdomain.com
# Should return 301 redirect to https://

# Verify TLS version
openssl s_client -connect jhe.yourdomain.com:443 -tls1_2
# Should connect successfully
```

#### Database SSL Connection

**For all cloud providers:**

```bash
# Test SSL connection
psql "postgresql://jheuser:password@db-host:5432/jhe_production?sslmode=require"
```

**AWS RDS:**

```bash
# Verify SSL is enforced
psql "postgresql://jheuser:password@jhe-production.xxxx.us-east-1.rds.amazonaws.com:5432/jhe_production?sslmode=require"

# Test with certificate verification
psql "postgresql://jheuser:password@jhe-production.xxxx.us-east-1.rds.amazonaws.com:5432/jhe_production?sslmode=verify-full&sslrootcert=/path/to/global-bundle.pem"

# Query SSL status from within database
psql -h jhe-production.xxxx.us-east-1.rds.amazonaws.com -U jheuser -d jhe_production -c "SELECT ssl_is_used();"
# Should return: t (true)
```

**Google Cloud SQL:**

```bash
# Get connection name
INSTANCE_CONNECTION_NAME=$(gcloud sql instances describe jhe-production --format='value(connectionName)')

# Test SSL connection
psql "host=/cloudsql/$INSTANCE_CONNECTION_NAME dbname=jhe_production user=jheuser sslmode=require"

# Or with IP
psql "host=INSTANCE_IP_ADDRESS dbname=jhe_production user=jheuser sslmode=verify-ca sslrootcert=/path/to/server-ca.pem"
```

**Azure:**

```bash
# Get server FQDN
SERVER_FQDN=$(az postgres flexible-server show \
  --resource-group jhe-rg \
  --name jhe-production \
  --query "fullyQualifiedDomainName" -o tsv)

# Test SSL connection
psql "host=$SERVER_FQDN dbname=jhe_production user=jheuser sslmode=require sslrootcert=/path/to/DigiCertGlobalRootCA.crt.pem"

# Verify SSL is enforced
az postgres flexible-server parameter show \
  --resource-group jhe-rg \
  --server-name jhe-production \
  --name require_secure_transport \
  --query value -o tsv
# Should return: on
```

### 2. Test Access Controls

Verify role-based access is working:

```python
# Test in Django shell
python manage.py shell

from core.models import Patient, Practitioner
from django.contrib.auth import get_user_model

# Verify RBAC permissions are enforced
practitioner_user = get_user_model().objects.get(email="practitioner@example.com")
patient = Patient.objects.get(id=1)

# This should only return patients the practitioner is authorized to see
authorized_patients = Patient.for_practitioner_organization_study(
    practitioner_user_id=practitioner_user.id,
    organization_id=1,
    study_id=1
)
```

Reference: `jupyterhealth-exchange/core/models.py`

### 3. Verify Consent Enforcement

Test that observations cannot be uploaded without consent:

```bash
# Attempt to upload observation without consent
curl -X POST https://jhe.yourdomain.com/fhir/r5/Observation \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "resourceType": "Observation",
    "subject": {"reference": "Patient/10001"},
    "code": {"coding": [{"system": "https://w3id.org/openmhealth", "code": "omh:blood-glucose:4.0"}]},
    "device": {"reference": "Device/70001"},
    "status": "final",
    "valueAttachment": {"data": "..."}
  }'

# Should return 403 Forbidden if consent not granted
```

Reference: `jupyterhealth-exchange/core/models.py`

## Backup and Disaster Recovery

### 1. Configure Automated Backups

For PostgreSQL with encryption:

```bash
# Create encrypted backup script
#!/bin/bash
BACKUP_FILE="/backups/jhe-$(date +%Y%m%d-%H%M%S).sql.gpg"
pg_dump -h $DB_HOST -U $DB_USER $DB_NAME | \
  gpg --encrypt --recipient admin@yourdomain.com > $BACKUP_FILE

# Verify backup was created
ls -lh $BACKUP_FILE
```

Schedule with cron:

```bash
0 2 * * * /opt/scripts/backup-jhe.sh
```

### 2. Test Restoration

Periodically test backup restoration:

```bash
# Decrypt and restore to test database
gpg --decrypt /backups/jhe-20250118-020000.sql.gpg | \
  psql -h test-db-host -U jheuser jhe_test
```

## Additional Security Measures

### 1. Configure Firewall Rules

Restrict database access to application servers only:

#### Self-Hosted (using ufw)

```bash
# Using ufw
sudo ufw allow from app-server-ip to any port 5432
sudo ufw deny 5432
```

#### AWS RDS Security Groups

```bash
# Create security group
aws ec2 create-security-group \
  --group-name jhe-db-sg \
  --description "JupyterHealth database access" \
  --vpc-id vpc-xxxxxxxx

# Get security group ID
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=jhe-db-sg" \
  --query 'SecurityGroups[0].GroupId' \
  --output text)

# Allow access from application server security group
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app-server-sg

# Or allow from specific IP
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 5432 \
  --cidr 10.0.1.0/24

# Apply to RDS instance
aws rds modify-db-instance \
  --db-instance-identifier jhe-production \
  --vpc-security-group-ids $SG_ID
```

#### Google Cloud SQL Authorized Networks

```bash
# Allow specific IP or CIDR range
gcloud sql instances patch jhe-production \
  --authorized-networks=10.0.1.0/24

# Or allow multiple networks
gcloud sql instances patch jhe-production \
  --authorized-networks=10.0.1.0/24,10.0.2.0/24

# For private IP (recommended - no public access)
gcloud sql instances patch jhe-production \
  --network=projects/PROJECT_ID/global/networks/NETWORK_NAME \
  --no-assign-ip

# Create private service connection
gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=RESERVED_RANGE_NAME \
  --network=NETWORK_NAME
```

#### Azure Database Firewall Rules

```bash
# Add firewall rule for specific IP
az postgres flexible-server firewall-rule create \
  --resource-group jhe-rg \
  --name jhe-production \
  --rule-name allow-app-server \
  --start-ip-address 10.0.1.5 \
  --end-ip-address 10.0.1.5

# Add rule for IP range
az postgres flexible-server firewall-rule create \
  --resource-group jhe-rg \
  --name jhe-production \
  --rule-name allow-app-subnet \
  --start-ip-address 10.0.1.0 \
  --end-ip-address 10.0.1.255

# For VNet integration (recommended)
az postgres flexible-server update \
  --resource-group jhe-rg \
  --name jhe-production \
  --vnet my-vnet \
  --subnet my-subnet \
  --public-access Disabled

# List all firewall rules
az postgres flexible-server firewall-rule list \
  --resource-group jhe-rg \
  --name jhe-production
```

### 2. Enable Database Connection Limits

In `postgresql.conf`:

```
max_connections = 100
```

In `.env`:

```bash
DB_CONN_MAX_AGE=600  # 10 minutes
```

### 3. Configure SAML2 SSO (Optional)

For enterprise deployments requiring SSO:

```bash
# In .env
SSO_VALID_DOMAINS="yourdomain.com,partner.com"
SAML_METADATA_URL="https://idp.yourdomain.com/metadata"
SAML_ACS_URL="https://jhe.yourdomain.com/saml2/acs/"
SAML_ENTITY_ID="https://jhe.yourdomain.com/saml2/metadata/"
```

Reference: `jupyterhealth-exchange/jhe/settings.py`

## Related Documentation

- [Troubleshooting Common Ingestion Errors](troubleshooting-ingestion-errors.md)
- [Upgrading JupyterHealth](upgrading-jupyterhealth.md)
- [Security Overview](../../explanation/security-overview.md)

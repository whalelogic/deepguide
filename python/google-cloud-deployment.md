# Google Cloud Platform Deployment for Flask

Complete guide to deploying Flask applications on Google Cloud Platform using Compute Engine VMs.

## Table of Contents
- [GCloud CLI Setup](#gcloud-cli-setup)
- [VM Instance Management](#vm-instance-management)
- [Flask Application Deployment](#flask-application-deployment)
- [Database Setup](#database-setup)
- [Production Configuration](#production-configuration)
- [Monitoring and Maintenance](#monitoring-and-maintenance)

---

## GCloud CLI Setup

### Installation

```bash
# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# macOS
brew install google-cloud-sdk

# Windows - Download installer
# https://cloud.google.com/sdk/docs/install

# Verify installation
gcloud --version
```

### Initial Configuration

```bash
# Initialize gcloud
gcloud init

# Login to Google Cloud
gcloud auth login

# Set default project
gcloud config set project YOUR_PROJECT_ID

# Set default region and zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# View current configuration
gcloud config list

# List available projects
gcloud projects list
```

---

## GCloud CLI Common Commands

| Command | Description |
|---------|-------------|
| `gcloud auth login` | Authenticate with Google Cloud |
| `gcloud config set project PROJECT_ID` | Set active project |
| `gcloud projects list` | List all projects |
| `gcloud compute instances list` | List all VM instances |
| `gcloud compute instances create NAME` | Create new VM instance |
| `gcloud compute instances start NAME` | Start VM instance |
| `gcloud compute instances stop NAME` | Stop VM instance |
| `gcloud compute instances delete NAME` | Delete VM instance |
| `gcloud compute ssh INSTANCE_NAME` | SSH into VM |
| `gcloud compute scp FILE INSTANCE:~/` | Copy file to VM |
| `gcloud compute firewall-rules list` | List firewall rules |
| `gcloud sql instances list` | List Cloud SQL instances |
| `gcloud app deploy` | Deploy to App Engine |

---

## VM Instance Management

### Create VM Instance

```bash
# Create basic VM for Flask app
gcloud compute instances create flask-app-vm \
    --zone=us-central1-a \
    --machine-type=e2-small \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=20GB \
    --boot-disk-type=pd-standard \
    --tags=http-server,https-server

# Create with startup script
gcloud compute instances create flask-app-vm \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=20GB \
    --metadata-from-file startup-script=startup.sh \
    --tags=http-server,https-server

# Create with static external IP
gcloud compute addresses create flask-app-ip --region=us-central1
gcloud compute instances create flask-app-vm \
    --address=flask-app-ip \
    --zone=us-central1-a \
    --machine-type=e2-small \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud
```

### Manage VM Instances

```bash
# List all instances
gcloud compute instances list

# Get instance details
gcloud compute instances describe flask-app-vm --zone=us-central1-a

# Start instance
gcloud compute instances start flask-app-vm --zone=us-central1-a

# Stop instance
gcloud compute instances stop flask-app-vm --zone=us-central1-a

# Restart instance
gcloud compute instances reset flask-app-vm --zone=us-central1-a

# Delete instance
gcloud compute instances delete flask-app-vm --zone=us-central1-a

# SSH into instance
gcloud compute ssh flask-app-vm --zone=us-central1-a

# Run command on instance
gcloud compute ssh flask-app-vm --zone=us-central1-a --command="ls -la"

# Copy files to instance
gcloud compute scp app.py flask-app-vm:~/app/ --zone=us-central1-a

# Copy files from instance
gcloud compute scp flask-app-vm:~/logs/app.log ./ --zone=us-central1-a

# Copy entire directory
gcloud compute scp --recurse ./flask-app flask-app-vm:~/ --zone=us-central1-a
```

### Firewall Rules

```bash
# Create firewall rule for Flask (port 5000)
gcloud compute firewall-rules create allow-flask \
    --allow=tcp:5000 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=http-server \
    --description="Allow Flask development server"

# Allow HTTP traffic (port 80)
gcloud compute firewall-rules create allow-http \
    --allow=tcp:80 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=http-server

# Allow HTTPS traffic (port 443)
gcloud compute firewall-rules create allow-https \
    --allow=tcp:443 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=https-server

# List firewall rules
gcloud compute firewall-rules list

# Delete firewall rule
gcloud compute firewall-rules delete allow-flask

# Update firewall rule
gcloud compute firewall-rules update allow-flask \
    --allow=tcp:5000,tcp:8000
```

---

## Flask Application Deployment

### Startup Script

```bash
# startup.sh
#!/bin/bash

# Update system
sudo apt-get update
sudo apt-get install -y python3-pip python3-venv nginx supervisor

# Create app directory
mkdir -p /home/flask_app
cd /home/flask_app

# Clone repository (or copy files)
# git clone https://github.com/username/flask-app.git .

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
pip install gunicorn

# Set environment variables
export FLASK_APP=app.py
export FLASK_ENV=production
export SECRET_KEY="your-production-secret-key"
export DATABASE_URL="sqlite:///instance/app.db"

# Initialize database
flask db upgrade

# Create systemd service
sudo tee /etc/systemd/system/flask-app.service > /dev/null <<EOF
[Unit]
Description=Flask Application
After=network.target

[Service]
User=flask_user
WorkingDirectory=/home/flask_app
Environment="PATH=/home/flask_app/venv/bin"
ExecStart=/home/flask_app/venv/bin/gunicorn -w 4 -b 127.0.0.1:8000 app:app

[Install]
WantedBy=multi-user.target
EOF

# Start service
sudo systemctl daemon-reload
sudo systemctl start flask-app
sudo systemctl enable flask-app
```

### Deploy Application

```bash
# 1. Copy application to VM
gcloud compute scp --recurse ./flask-app flask-app-vm:~/ --zone=us-central1-a

# 2. SSH into VM
gcloud compute ssh flask-app-vm --zone=us-central1-a

# 3. Setup application (on VM)
cd flask-app
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 4. Configure environment
cat > .env <<EOF
FLASK_APP=app.py
FLASK_ENV=production
SECRET_KEY=$(python -c 'import secrets; print(secrets.token_hex(32))')
DATABASE_URL=sqlite:///instance/app.db
EOF

# 5. Initialize database
flask db upgrade

# 6. Test application
gunicorn -b 0.0.0.0:8000 app:app
```

### Nginx Configuration

```bash
# /etc/nginx/sites-available/flask-app
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;
    
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    location /static {
        alias /home/flask_app/static;
        expires 30d;
    }
    
    location /media {
        alias /home/flask_app/media;
        expires 30d;
    }
}

# Enable site
sudo ln -s /etc/nginx/sites-available/flask-app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## Systemd Service Management

### Create Systemd Service

```bash
# /etc/systemd/system/flask-app.service
[Unit]
Description=Flask Application
After=network.target

[Service]
User=flask_user
Group=www-data
WorkingDirectory=/home/flask_app
Environment="PATH=/home/flask_app/venv/bin"
Environment="FLASK_ENV=production"
ExecStart=/home/flask_app/venv/bin/gunicorn \
    --workers 4 \
    --bind 127.0.0.1:8000 \
    --access-logfile /home/flask_app/logs/access.log \
    --error-logfile /home/flask_app/logs/error.log \
    app:app

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Service Commands

```bash
# Reload systemd
sudo systemctl daemon-reload

# Start service
sudo systemctl start flask-app

# Stop service
sudo systemctl stop flask-app

# Restart service
sudo systemctl restart flask-app

# Enable on boot
sudo systemctl enable flask-app

# Disable on boot
sudo systemctl disable flask-app

# Check status
sudo systemctl status flask-app

# View logs
sudo journalctl -u flask-app -f

# View recent logs
sudo journalctl -u flask-app -n 100 --no-pager
```

---

## Production Configuration

### Environment Variables

```python
# config.py
import os
from datetime import timedelta

class ProductionConfig:
    # Flask
    SECRET_KEY = os.environ.get('SECRET_KEY')
    DEBUG = False
    TESTING = False
    
    # Database
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL')
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_POOL_SIZE = 10
    SQLALCHEMY_POOL_RECYCLE = 3600
    
    # Session
    SESSION_TYPE = 'redis'
    SESSION_REDIS = redis.from_url(os.environ.get('REDIS_URL'))
    PERMANENT_SESSION_LIFETIME = timedelta(days=7)
    SESSION_COOKIE_SECURE = True
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SAMESITE = 'Lax'
    
    # Security
    WTF_CSRF_ENABLED = True
    WTF_CSRF_TIME_LIMIT = None
    
    # Mail
    MAIL_SERVER = os.environ.get('MAIL_SERVER')
    MAIL_PORT = int(os.environ.get('MAIL_PORT', 587))
    MAIL_USE_TLS = True
    MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
    MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')
    
    # Redis
    REDIS_URL = os.environ.get('REDIS_URL', 'redis://localhost:6379/0')
    
    # Celery
    CELERY_BROKER_URL = os.environ.get('REDIS_URL')
    CELERY_RESULT_BACKEND = os.environ.get('REDIS_URL')
```

### Set Environment Variables on VM

```bash
# SSH into VM
gcloud compute ssh flask-app-vm --zone=us-central1-a

# Create environment file
sudo nano /etc/environment

# Add variables
SECRET_KEY=your-production-secret-key
DATABASE_URL=sqlite:///instance/production.db
REDIS_URL=redis://localhost:6379/0
MAIL_SERVER=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password

# Or use .env file
cd /home/flask_app
nano .env
# Add variables then load them
set -a
source .env
set +a
```

---

## Cloud SQL Integration

### Create Cloud SQL Instance

```bash
# Create PostgreSQL instance
gcloud sql instances create flask-db \
    --database-version=POSTGRES_14 \
    --tier=db-f1-micro \
    --region=us-central1

# Set root password
gcloud sql users set-password root \
    --instance=flask-db \
    --password=YOUR_PASSWORD

# Create database
gcloud sql databases create flask_app --instance=flask-db

# Get connection name
gcloud sql instances describe flask-db | grep connectionName

# Connect VM to Cloud SQL
gcloud compute instances add-access-config flask-app-vm
```

### Connect Flask to Cloud SQL

```python
# For Cloud SQL Proxy
# Install proxy on VM
wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
chmod +x cloud_sql_proxy

# Run proxy
./cloud_sql_proxy -instances=PROJECT:REGION:INSTANCE=tcp:5432 &

# Update Flask config
SQLALCHEMY_DATABASE_URI = 'postgresql://user:password@localhost/flask_app'
```

---

## SSL/TLS Configuration

### Install Certbot (Let's Encrypt)

```bash
# SSH into VM
gcloud compute ssh flask-app-vm --zone=us-central1-a

# Install certbot
sudo apt-get update
sudo apt-get install -y certbot python3-certbot-nginx

# Get SSL certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Test automatic renewal
sudo certbot renew --dry-run

# Renew certificates (automatic cron job created)
sudo certbot renew
```

### Updated Nginx with SSL

```nginx
# /etc/nginx/sites-available/flask-app
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;
    
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    location /static {
        alias /home/flask_app/static;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## Deployment Automation

### Deployment Script

```bash
#!/bin/bash
# deploy.sh

set -e

PROJECT_ID="your-project-id"
INSTANCE_NAME="flask-app-vm"
ZONE="us-central1-a"
APP_DIR="/home/flask_app"

echo "🚀 Deploying Flask application..."

# Copy files to VM
echo "📦 Copying files..."
gcloud compute scp --recurse \
    --zone=$ZONE \
    --project=$PROJECT_ID \
    ./app \
    ./requirements.txt \
    ./config.py \
    $INSTANCE_NAME:$APP_DIR/

# SSH and run deployment commands
echo "⚙️  Installing dependencies and restarting service..."
gcloud compute ssh $INSTANCE_NAME \
    --zone=$ZONE \
    --project=$PROJECT_ID \
    --command="
        cd $APP_DIR && \
        source venv/bin/activate && \
        pip install -r requirements.txt && \
        flask db upgrade && \
        sudo systemctl restart flask-app && \
        sudo systemctl restart nginx
    "

echo "✅ Deployment complete!"

# Check service status
gcloud compute ssh $INSTANCE_NAME \
    --zone=$ZONE \
    --project=$PROJECT_ID \
    --command="sudo systemctl status flask-app --no-pager"
```

### Rolling Update Script

```bash
#!/bin/bash
# rolling-update.sh

INSTANCES=("flask-app-vm-1" "flask-app-vm-2" "flask-app-vm-3")
ZONE="us-central1-a"

for INSTANCE in "${INSTANCES[@]}"; do
    echo "Updating $INSTANCE..."
    
    # Remove from load balancer
    gcloud compute backend-services remove-backend flask-backend \
        --instance=$INSTANCE \
        --instance-zone=$ZONE
    
    # Deploy update
    ./deploy.sh $INSTANCE
    
    # Health check
    sleep 10
    
    # Add back to load balancer
    gcloud compute backend-services add-backend flask-backend \
        --instance=$INSTANCE \
        --instance-zone=$ZONE
    
    echo "$INSTANCE updated successfully"
    sleep 30  # Wait before updating next instance
done
```

---

## Monitoring and Maintenance

### Logging

```python
# logging_config.py
import logging
from logging.handlers import RotatingFileHandler
import os

def setup_logging(app):
    """Configure application logging."""
    
    if not app.debug:
        # Create logs directory
        if not os.path.exists('logs'):
            os.mkdir('logs')
        
        # File handler
        file_handler = RotatingFileHandler(
            'logs/flask_app.log',
            maxBytes=10240000,  # 10 MB
            backupCount=10
        )
        file_handler.setFormatter(logging.Formatter(
            '%(asctime)s %(levelname)s: %(message)s '
            '[in %(pathname)s:%(lineno)d]'
        ))
        file_handler.setLevel(logging.INFO)
        app.logger.addHandler(file_handler)
        
        app.logger.setLevel(logging.INFO)
        app.logger.info('Flask application startup')
```

### View Logs

```bash
# Application logs
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="tail -f /home/flask_app/logs/flask_app.log"

# System logs
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="sudo journalctl -u flask-app -f"

# Nginx access logs
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="sudo tail -f /var/log/nginx/access.log"

# Nginx error logs
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="sudo tail -f /var/log/nginx/error.log"
```

### Monitoring Commands

```bash
# Check instance status
gcloud compute instances list

# Get instance metrics
gcloud compute instances describe flask-app-vm \
    --zone=us-central1-a \
    --format="value(status, networkInterfaces[0].accessConfigs[0].natIP)"

# Monitor CPU usage
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="top -bn1 | head -20"

# Monitor disk usage
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="df -h"

# Monitor memory
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="free -h"

# Check running processes
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="ps aux | grep gunicorn"

# Check open ports
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="sudo netstat -tlnp"
```

---

## Database Backup

### Automated Backup Script

```bash
#!/bin/bash
# backup-db.sh

INSTANCE_NAME="flask-app-vm"
ZONE="us-central1-a"
BACKUP_BUCKET="gs://your-backup-bucket"
DATE=$(date +%Y%m%d_%H%M%S)

echo "🔄 Starting database backup..."

# Create backup on VM
gcloud compute ssh $INSTANCE_NAME --zone=$ZONE --command="
    cd /home/flask_app && \
    sqlite3 instance/app.db \".backup 'instance/backup_$DATE.db'\" && \
    gzip instance/backup_$DATE.db
"

# Copy to Cloud Storage
gcloud compute scp \
    $INSTANCE_NAME:/home/flask_app/instance/backup_$DATE.db.gz \
    ./backup_$DATE.db.gz \
    --zone=$ZONE

gsutil cp ./backup_$DATE.db.gz $BACKUP_BUCKET/

# Cleanup local backup
rm ./backup_$DATE.db.gz

# Cleanup old backups on VM
gcloud compute ssh $INSTANCE_NAME --zone=$ZONE --command="
    find /home/flask_app/instance/backup_*.db.gz -mtime +7 -delete
"

echo "✅ Backup complete: backup_$DATE.db.gz"
```

### Schedule Backups with Cron

```bash
# SSH into VM
gcloud compute ssh flask-app-vm --zone=us-central1-a

# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /home/flask_app/scripts/backup-db.sh >> /home/flask_app/logs/backup.log 2>&1
```

---

## Load Balancer Setup

### Create Load Balancer

```bash
# Create instance template
gcloud compute instance-templates create flask-app-template \
    --machine-type=e2-small \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --metadata-from-file startup-script=startup.sh \
    --tags=http-server

# Create instance group
gcloud compute instance-groups managed create flask-app-group \
    --base-instance-name=flask-app \
    --size=3 \
    --template=flask-app-template \
    --zone=us-central1-a

# Create health check
gcloud compute health-checks create http flask-health-check \
    --port=8000 \
    --request-path=/health

# Create backend service
gcloud compute backend-services create flask-backend \
    --protocol=HTTP \
    --health-checks=flask-health-check \
    --global

# Add instance group to backend
gcloud compute backend-services add-backend flask-backend \
    --instance-group=flask-app-group \
    --instance-group-zone=us-central1-a \
    --global

# Create URL map
gcloud compute url-maps create flask-lb \
    --default-service=flask-backend

# Create HTTP proxy
gcloud compute target-http-proxies create flask-http-proxy \
    --url-map=flask-lb

# Create forwarding rule
gcloud compute forwarding-rules create flask-http-rule \
    --global \
    --target-http-proxy=flask-http-proxy \
    --ports=80

# Get load balancer IP
gcloud compute forwarding-rules describe flask-http-rule --global
```

---

## Scaling

### Auto-scaling Configuration

```bash
# Set autoscaling policy
gcloud compute instance-groups managed set-autoscaling flask-app-group \
    --max-num-replicas=10 \
    --min-num-replicas=2 \
    --target-cpu-utilization=0.7 \
    --cool-down-period=90 \
    --zone=us-central1-a

# Update instance group size manually
gcloud compute instance-groups managed resize flask-app-group \
    --size=5 \
    --zone=us-central1-a
```

---

## Deployment Checklist

### Pre-Deployment

- [ ] Set `DEBUG = False` in production config
- [ ] Use strong `SECRET_KEY`
- [ ] Configure proper database (not SQLite for high traffic)
- [ ] Set up Redis for caching and sessions
- [ ] Enable HTTPS with SSL certificate
- [ ] Configure firewall rules properly
- [ ] Set up monitoring and alerting
- [ ] Create backup strategy
- [ ] Use environment variables for secrets
- [ ] Set up logging

### Post-Deployment

- [ ] Test all critical endpoints
- [ ] Verify database connectivity
- [ ] Check Redis connection
- [ ] Test email delivery
- [ ] Monitor error logs
- [ ] Verify SSL certificate
- [ ] Test auto-scaling (if configured)
- [ ] Set up backup automation
- [ ] Configure monitoring alerts
- [ ] Document deployment process

---

## Troubleshooting

### Common Issues

```bash
# Application won't start
gcloud compute ssh flask-app-vm --zone=us-central1-a
sudo journalctl -u flask-app -n 50 --no-pager

# Port not accessible
gcloud compute firewall-rules list
curl http://EXTERNAL_IP:5000

# Database errors
gcloud compute ssh flask-app-vm --zone=us-central1-a
cd /home/flask_app
source venv/bin/activate
flask shell
>>> db.session.execute('SELECT 1')

# Nginx not working
gcloud compute ssh flask-app-vm --zone=us-central1-a
sudo nginx -t
sudo systemctl status nginx

# Check disk space
gcloud compute ssh flask-app-vm --zone=us-central1-a --command="df -h"

# Check memory
gcloud compute ssh flask-app-vm --zone=us-central1-a --command="free -h"

# View application logs
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="tail -100 /home/flask_app/logs/flask_app.log"
```

---

## Cost Optimization

```bash
# Use preemptible instances for dev/test
gcloud compute instances create flask-dev-vm \
    --preemptible \
    --machine-type=e2-micro \
    --zone=us-central1-a

# Stop instances during non-business hours
# Create Cloud Scheduler job
gcloud scheduler jobs create http stop-vm-job \
    --schedule="0 18 * * 1-5" \
    --uri="https://compute.googleapis.com/compute/v1/projects/PROJECT/zones/ZONE/instances/INSTANCE/stop" \
    --http-method=POST

# Downgrade machine type
gcloud compute instances stop flask-app-vm --zone=us-central1-a
gcloud compute instances set-machine-type flask-app-vm \
    --machine-type=e2-micro \
    --zone=us-central1-a
gcloud compute instances start flask-app-vm --zone=us-central1-a

# Monitor costs
gcloud billing accounts list
gcloud billing projects describe PROJECT_ID
```

---

## Quick Reference

### Essential Commands

```bash
# Deploy application
./deploy.sh

# SSH into instance
gcloud compute ssh flask-app-vm --zone=us-central1-a

# Restart application
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="sudo systemctl restart flask-app"

# View logs
gcloud compute ssh flask-app-vm --zone=us-central1-a \
    --command="sudo journalctl -u flask-app -f"

# Backup database
./backup-db.sh

# Update instance
gcloud compute instances stop flask-app-vm --zone=us-central1-a
# Make changes
gcloud compute instances start flask-app-vm --zone=us-central1-a

# Check instance status
gcloud compute instances list --filter="name=flask-app-vm"
```

---

## Related Documentation

- [Google Cloud Compute Engine](https://cloud.google.com/compute/docs)
- [GCloud CLI Reference](https://cloud.google.com/sdk/gcloud/reference)
- [Cloud SQL Documentation](https://cloud.google.com/sql/docs)
- [Google Cloud Console](https://console.cloud.google.com/)

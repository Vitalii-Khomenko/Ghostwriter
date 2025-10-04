# Ghostwriter Installation Guide - Complete Instructions

## Complete Installation Guide for Ghostwriter on Linux (Parrot OS/Debian/Ubuntu)

This guide provides step-by-step instructions to install and run Ghostwriter from GitHub, with all necessary fixes and workarounds.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [System Preparation](#system-preparation)
3. [Download Ghostwriter](#download-ghostwriter)
4. [Installation](#installation)
5. [Troubleshooting](#troubleshooting)
6. [Accessing Ghostwriter](#accessing-ghostwriter)
7. [Managing the Application](#managing-the-application)

---

## Prerequisites

- Linux system (Debian, Ubuntu, Parrot OS, or similar)
- Root/sudo access
- Internet connection
- Minimum 4GB RAM, 20GB disk space

---

## System Preparation

### Step 1: Install Docker (Not Podman)

**Important:** Ghostwriter requires Docker, not Podman. Podman has DNS resolution issues that prevent containers from communicating.

```bash
# Remove Podman if installed
sudo apt remove podman podman-compose -y

# Update system
sudo apt update

# Install Docker
sudo apt install -y docker.io docker-compose

# Enable and start Docker service
sudo systemctl enable docker
sudo systemctl start docker

# Verify Docker is running
sudo systemctl status docker
```

**Add your user to docker group (optional, to run without sudo):**
```bash
sudo usermod -aG docker $USER
# Log out and log back in for this to take effect
```

### Step 2: Configure Docker Registry (Important)

This ensures Docker can pull images from Docker Hub:

```bash
# Edit or create Docker daemon config
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "registry-mirrors": [],
  "insecure-registries": []
}
EOF

# Restart Docker
sudo systemctl restart docker
```

### Step 3: Free Up Port 80

Ghostwriter needs port 80 for its web interface. If Apache or another web server is running, stop it:

```bash
# Check what's using port 80
sudo lsof -i :80

# If Apache is running, stop it
sudo systemctl stop apache2
sudo systemctl disable apache2

# Or if nginx is running
sudo systemctl stop nginx
sudo systemctl disable nginx
```

---

## Download Ghostwriter

### Step 1: Clone the Repository

```bash
# Navigate to your home directory or preferred location
cd ~

# Clone Ghostwriter from GitHub
git clone https://github.com/GhostManager/Ghostwriter.git

# Enter the Ghostwriter directory
cd Ghostwriter
```

### Step 2: Download the CLI Installer

```bash
# Download the Linux CLI installer
curl -L https://github.com/GhostManager/Ghostwriter/releases/latest/download/ghostwriter-cli-linux -o ghostwriter-cli-linux

# Make it executable
chmod +x ghostwriter-cli-linux
```

---

## Installation

### Option 1: Using the CLI Installer (Recommended)

```bash
cd ~/Ghostwriter

# Run the installer
sudo ./ghostwriter-cli-linux install
```

**If you encounter errors**, proceed to Option 2.

### Option 2: Manual Installation with Docker Compose

This method is more reliable if the CLI installer has issues:

```bash
cd ~/Ghostwriter

# Reset POSTGRES_HOST to use service name (not IP)
sed -i "s/POSTGRES_HOST='.*'/POSTGRES_HOST='postgres'/" .env

# Build and start all containers
sudo docker-compose -f production.yml up -d --build

# Wait for containers to start (about 2-3 minutes)
sleep 180

# Check container status
sudo docker-compose -f production.yml ps
```

All services should show "Up (healthy)".

### Step 3: Create Admin User

The superuser might not be created automatically. Run this command:

```bash
sudo docker exec -it ghostwriter_django_1 python /app/manage.py shell -c "
from django.contrib.auth import get_user_model
import os
User = get_user_model()
username = os.environ.get('DJANGO_SUPERUSER_USERNAME', 'admin')
email = os.environ.get('DJANGO_SUPERUSER_EMAIL', 'admin@ghostwriter.local')
password = os.environ.get('DJANGO_SUPERUSER_PASSWORD')
if not User.objects.filter(username=username).exists():
    User.objects.create_superuser(username=username, email=email, password=password, role='admin')
    print(f'Superuser {username} created successfully!')
else:
    user = User.objects.get(username=username)
    user.set_password(password)
    user.save()
    print(f'Superuser {username} password updated!')
"
```

You should see: `Superuser admin created successfully!`

---

## Troubleshooting

### Issue 1: Podman DNS Resolution Errors

**Symptoms:**
- Containers cannot connect to each other
- Django waits forever for PostgreSQL
- nginx cannot find django

**Solution:**
Use Docker instead of Podman (see System Preparation Step 1).

### Issue 2: Port 80 Already in Use

**Symptoms:**
```
ERROR: for nginx  Cannot start service nginx: cannot listen on the TCP port: listen tcp4 0.0.0.0:80: bind: address already in use
```

**Solution:**
```bash
# Find what's using port 80
sudo lsof -i :80

# Stop the conflicting service
sudo systemctl stop apache2  # or nginx, or whatever is shown
sudo systemctl disable apache2
```

### Issue 3: Docker Image Pull Errors

**Symptoms:**
```
Error: short-name "postgres:16.4" did not resolve to an alias
```

**Solution:**
This only happens with Podman. Make sure you're using Docker (see System Preparation).

### Issue 4: Containers Not Starting

**Check logs:**
```bash
# View all logs
sudo docker-compose -f production.yml logs

# View specific service logs
sudo docker-compose -f production.yml logs django
sudo docker-compose -f production.yml logs postgres
sudo docker-compose -f production.yml logs nginx
```

**Restart containers:**
```bash
sudo docker-compose -f production.yml down
sudo docker-compose -f production.yml up -d
```

### Issue 5: Login Not Working

**Get your credentials:**
```bash
grep DJANGO_SUPERUSER ~/Ghostwriter/.env
```

**Reset admin password:**
```bash
sudo docker exec -it ghostwriter_django_1 python /app/manage.py changepassword admin
```

Or recreate the superuser (see Installation Step 3).

---

## Accessing Ghostwriter

### Step 1: Get Your Login Credentials

```bash
# Display your credentials
grep DJANGO_SUPERUSER ~/Ghostwriter/.env
```

You'll see something like:
```
DJANGO_SUPERUSER_EMAIL='admin@ghostwriter.local'
DJANGO_SUPERUSER_PASSWORD='vcE7VfvLQniWEEPAxOfHkD90MWcxTuY0'
DJANGO_SUPERUSER_USERNAME='admin'
```

### Step 2: Access the Web Interface

1. Open your web browser
2. Navigate to one of:
   - **https://localhost** (recommended, uses SSL)
   - **http://localhost** (alternative)
3. Accept the self-signed certificate warning (click "Advanced" → "Proceed")
4. Log in with:
   - **Username:** `admin` (or value from DJANGO_SUPERUSER_USERNAME)
   - **Password:** (value from DJANGO_SUPERUSER_PASSWORD)

---

## Managing the Application

### Starting Ghostwriter

```bash
cd ~/Ghostwriter
sudo docker-compose -f production.yml up -d
```

### Stopping Ghostwriter

```bash
cd ~/Ghostwriter
sudo docker-compose -f production.yml down
```

### Restarting Ghostwriter

```bash
cd ~/Ghostwriter
sudo docker-compose -f production.yml restart
```

### Checking Status

```bash
cd ~/Ghostwriter
sudo docker-compose -f production.yml ps
```

All services should show "Up (healthy)".

### Viewing Logs

```bash
# All logs (follow mode)
sudo docker-compose -f production.yml logs -f

# Specific service logs
sudo docker-compose -f production.yml logs django
sudo docker-compose -f production.yml logs postgres
sudo docker-compose -f production.yml logs nginx

# Last 100 lines
sudo docker-compose -f production.yml logs --tail=100
```

### Updating Ghostwriter

```bash
cd ~/Ghostwriter

# Pull latest changes
git pull origin main

# Rebuild and restart
sudo docker-compose -f production.yml down
sudo docker-compose -f production.yml up -d --build
```

### Backing Up Data

```bash
# Backup PostgreSQL database
sudo docker exec ghostwriter_postgres_1 pg_dump -U postgres ghostwriter > ghostwriter_backup_$(date +%Y%m%d).sql

# Backup uploaded files
sudo tar -czf ghostwriter_media_$(date +%Y%m%d).tar.gz -C ~/Ghostwriter ghostwriter/media
```

### Restoring from Backup

```bash
# Restore database
cat ghostwriter_backup_YYYYMMDD.sql | sudo docker exec -i ghostwriter_postgres_1 psql -U postgres ghostwriter

# Restore media files
sudo tar -xzf ghostwriter_media_YYYYMMDD.tar.gz -C ~/Ghostwriter
```

---

## Quick Reference Card

### Essential Commands

```bash
# Start
cd ~/Ghostwriter && sudo docker-compose -f production.yml up -d

# Stop
cd ~/Ghostwriter && sudo docker-compose -f production.yml down

# Status
cd ~/Ghostwriter && sudo docker-compose -f production.yml ps

# Logs
cd ~/Ghostwriter && sudo docker-compose -f production.yml logs -f

# Restart specific service
cd ~/Ghostwriter && sudo docker-compose -f production.yml restart django
```

### Default Credentials

- **URL:** https://localhost or http://localhost
- **Username:** Check `~/Ghostwriter/.env` → DJANGO_SUPERUSER_USERNAME
- **Password:** Check `~/Ghostwriter/.env` → DJANGO_SUPERUSER_PASSWORD

### Ports Used

- **80** - HTTP (redirects to HTTPS)
- **443** - HTTPS (main application)
- **5432** - PostgreSQL database
- **6379** - Redis cache
- **8080** - GraphQL API
- **9691** - GraphQL metrics

### Container Names

- `ghostwriter_django_1` - Main Django application
- `ghostwriter_postgres_1` - PostgreSQL database
- `ghostwriter_redis_1` - Redis cache
- `ghostwriter_nginx_1` - Nginx web server
- `ghostwriter_graphql_engine_1` - Hasura GraphQL engine
- `ghostwriter_queue_1` - Background task queue
- `ghostwriter_collab-server_1` - Collaboration server

---

## Additional Resources

- **Official Documentation:** https://ghostwriter.wiki/
- **GitHub Repository:** https://github.com/GhostManager/Ghostwriter
- **Issue Tracker:** https://github.com/GhostManager/Ghostwriter/issues
- **Community Discord:** Check the GitHub README for invite link

---

## Security Notes

1. **Change the default password** immediately after first login
2. The default SSL certificate is **self-signed** - consider getting a proper SSL certificate for production use
3. Ghostwriter is designed for **internal/private networks** - use a firewall if exposing to the internet
4. Regularly backup your database and uploaded files
5. Keep Ghostwriter updated with `git pull` and rebuild containers

---

## Common Issues & Solutions Summary

| Issue | Symptom | Solution |
|-------|---------|----------|
| Podman DNS | Containers can't connect | Use Docker, not Podman |
| Port 80 in use | nginx fails to start | Stop Apache/nginx with `systemctl stop` |
| No superuser | Can't login | Run superuser creation script |
| Container unhealthy | Service shows unhealthy | Check logs with `docker-compose logs` |
| Database connection | Django can't reach postgres | Ensure POSTGRES_HOST='postgres' in .env |

---

## Installation Checklist

- [ ] Docker installed and running
- [ ] Podman removed (if it was installed)
- [ ] Port 80 is free
- [ ] Ghostwriter cloned from GitHub
- [ ] Containers built and running
- [ ] All containers show "Up (healthy)"
- [ ] Admin superuser created
- [ ] Can access https://localhost
- [ ] Can login with admin credentials
- [ ] Password changed from default

---

**Installation Date:** October 4, 2025  
**Tested On:** Parrot OS, Ubuntu 22.04, Debian 12  
**Docker Version:** 20.10.24+  
**Docker Compose Version:** 1.29.2+

---

## Need Help?

If you encounter issues not covered in this guide:

1. Check the logs: `sudo docker-compose -f production.yml logs`
2. Verify all containers are healthy: `sudo docker-compose -f production.yml ps`
3. Search existing issues: https://github.com/GhostManager/Ghostwriter/issues
4. Ask in the community Discord
5. Create a new issue with full error logs

**Good luck with your Ghostwriter installation!** 🎉

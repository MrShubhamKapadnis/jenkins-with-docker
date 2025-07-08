# Jenkins Docker Setup - Complete Knowledge Transfer Documentation
# Begineer to Advance configurations
## Document Overview
This document provides a complete, self-explanatory guide for setting up Jenkins using Docker with persistent data storage. It serves as both a setup guide and knowledge transfer document for everyone.

**Document Version**: 1.0  
**Last Updated**: July 2025  
**Author**: Shubham Kapadnis
**Purpose**: Jenkins complete setup with Docker containerization

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Environment Setup](#environment-setup)
4. [Installation Methods](#installation-methods)
5. [Post-Installation Configuration](#post-installation-configuration)
6. [Operational Procedures](#operational-procedures)
7. [Troubleshooting Guide](#troubleshooting-guide)
8. [Security Considerations](#security-considerations)
9. [Maintenance & Backup](#maintenance--backup)
10. [Knowledge Transfer Checklist](#knowledge-transfer-checklist)

---

## Architecture Overview

### Why This Setup?
- **Containerization**: Jenkins runs in Docker for isolation and portability
- **Data Persistence**: All data stored on mounted disk (`/home`) to prevent data loss
- **Scalability**: Easy to backup, restore, and migrate
- **Security**: Isolated environment with controlled access

### Components
```
┌─────────────────────────────────────────┐
│                 HOST VM                 │
│  ┌─────────────────────────────────────┐│
│  │         Docker Engine              ││
│  │  ┌─────────────────────────────────┐││
│  │  │      Jenkins Container          │││
│  │  │                                 │││
│  │  │  Port 8080 → Web Interface      │││
│  │  │  Port 50000 → Agent Connection  │││
│  │  │                                 │││
│  │  │  Volume: /var/jenkins_home      │││
│  │  │     ↓                          │││
│  │  └─────────────────────────────────┘││
│  └─────────────────────────────────────┘│
│                    ↓                    │
│  ┌─────────────────────────────────────┐│
│  │     /home/jenkins_data/             ││
│  │  (Persistent Storage on Mounted     ││
│  │   Disk - NOT on Root Filesystem)    ││
│  └─────────────────────────────────────┘│
└─────────────────────────────────────────┘
```

---

## Prerequisites

### System Requirements
- **OS**: Linux (CentOS/RHEL/Ubuntu)
- **RAM**: Minimum 2GB, Recommended 4GB+
- **Disk**: `/home` mounted with at least 10GB free space
- **Network**: Ports 8080 and 50000 available

### Software Prerequisites
```bash
# Check if Docker is installed
docker --version
# Expected output: Docker version 20.x.x or higher

# Check if Docker is running
sudo systemctl status docker

# Check mounted disk space
df -h /home
# Should show sufficient free space
```

### Pre-Installation Checklist
- [ ] Docker installed and running
- [ ] User has sudo privileges
- [ ] Firewall configured (ports 8080, 50000)
- [ ] `/home` directory is mounted and writable
- [ ] Network connectivity for Docker image pulls

---

## Environment Setup

### Step 1: Create Directory Structure
```bash
# Create Jenkins data directory on mounted disk
sudo mkdir -p /home/jenkins_data
sudo mkdir -p /home/jenkins_data/jenkins_home

# Set proper ownership (Jenkins runs as user ID 1000)
sudo chown -R 1000:1000 /home/jenkins_data

# Verify directory creation
ls -la /home/jenkins_data/
```

**Expected Output:**
```
drwxr-xr-x. 3 1000 1000 4096 Jul  7 10:00 jenkins_home
```

### Step 2: Docker Group Setup
```bash
# Create docker group if it doesn't exist
sudo groupadd docker 2>/dev/null || true

# Add current user to docker group (optional)
sudo usermod -aG docker $USER

# Verify docker group exists
getent group docker
```

---

## Installation Methods

### Method 1: Docker Compose (Recommended)

**Why Docker Compose?**
- Declarative configuration
- Easy to version control
- Simplified management
- Environment reproducibility

#### Create Docker Compose File
```bash
# Navigate to Jenkins directory
cd /home/jenkins_data

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"   # Web interface
      - "50000:50000" # Agent connections
    volumes:
      - /home/jenkins_data/jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JENKINS_OPTS=--httpPort=8080
      - JAVA_OPTS=-Xmx2048m -Xms512m
    user: root
    networks:
      - jenkins-network

networks:
  jenkins-network:
    driver: bridge
EOF
```

#### Start Jenkins
```bash
# Start Jenkins container
docker compose up -d

# Verify container is running
docker compose ps
```

**Expected Output:**
```
NAME                COMMAND                  SERVICE             STATUS              PORTS
jenkins             "/sbin/tini -- /usr/…"   jenkins             running             0.0.0.0:8080->8080/tcp, 0.0.0.0:50000->50000/tcp
```

### Method 2: Docker Run Command

**Use Case**: Quick setup or when Docker Compose is not available

```bash
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v /home/jenkins_data/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --user root \
  -e JAVA_OPTS="-Xmx2048m -Xms512m" \
  jenkins/jenkins:lts
```

### Method 3: Custom Jenkins Image (Advanced)

**Use Case**: When you need additional tools pre-installed

```bash
# Create custom Dockerfile
cat > /home/jenkins_data/Dockerfile << 'EOF'
FROM jenkins/jenkins:lts

USER root

# Install additional tools
RUN apt-get update && \
    apt-get install -y \
        curl \
        wget \
        git \
        vim \
        apt-transport-https \
        ca-certificates \
        gnupg \
        lsb-release && \
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg && \
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*

# Install common build tools
RUN apt-get update && \
    apt-get install -y \
        build-essential \
        python3 \
        python3-pip \
        nodejs \
        npm && \
    rm -rf /var/lib/apt/lists/*

USER jenkins

# Pre-install common plugins
RUN jenkins-plugin-cli --plugins \
    git \
    workflow-aggregator \
    docker-workflow \
    blueocean \
    pipeline-stage-view
EOF

# Build custom image
docker build -t jenkins-custom /home/jenkins_data/

# Run with custom image
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v /home/jenkins_data/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --user root \
  jenkins-custom
```

---

## Post-Installation Configuration

### Step 1: Initial Jenkins Setup

#### Access Jenkins Web Interface
```bash
# Get your VM's IP address
ip addr show | grep "inet " | grep -v 127.0.0.1

# Access Jenkins at: http://YOUR_VM_IP:8080
```

#### Retrieve Initial Admin Password
```bash
# Method 1: From container logs
docker logs jenkins 2>&1 | grep -A 5 -B 5 "initialAdminPassword"

# Method 2: From file system
sudo cat /home/jenkins_data/jenkins_home/secrets/initialAdminPassword

# Method 3: Execute command in container
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Step 2: Jenkins Setup Wizard

1. **Unlock Jenkins**
   - Enter the initial admin password
   - Click "Continue"

2. **Customize Jenkins**
   - Choose "Install suggested plugins" (Recommended)
   - Wait for plugin installation (5-10 minutes)

3. **Create First Admin User**
   ```
   Username: admin
   Password: [secure-password]
   Full Name: Jenkins Administrator
   Email: admin@yourcompany.com
   ```

4. **Instance Configuration**
   - Jenkins URL: `http://YOUR_VM_IP:8080/`
   - Click "Save and Finish"

### Step 3: Essential Plugin Installation

Navigate to **Manage Jenkins** → **Manage Plugins** → **Available**

#### Recommended Plugins
- **Docker Plugin**: For Docker integration
- **Docker Pipeline Plugin**: For Docker in pipelines
- **Blue Ocean**: Modern UI for pipelines
- **Git Plugin**: Git integration
- **Pipeline Plugin**: Pipeline as code
- **Workspace Cleanup Plugin**: Clean workspaces
- **Build Timeout Plugin**: Prevent hanging builds
- **Timestamper Plugin**: Add timestamps to logs
- **AnsiColor Plugin**: Colored console output

#### Installation Commands
```bash
# Install plugins via Jenkins CLI (optional)
docker exec jenkins jenkins-plugin-cli --plugins \
  docker-workflow \
  blueocean \
  git \
  pipeline-stage-view \
  build-timeout \
  timestamper \
  ansicolor \
  ws-cleanup
```

### Step 4: Configure Docker Integration

#### Test Docker Access
1. Go to **Manage Jenkins** → **Script Console**
2. Run this Groovy script:

```groovy
def sout = new StringBuffer()
def serr = new StringBuffer()
def proc = 'docker version'.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println "Docker Version Output:"
println "STDOUT: $sout"
println "STDERR: $serr"
```

**Expected Output**: Should show Docker version information

---

## Operational Procedures

### Daily Operations

#### Starting Jenkins
```bash
# Using Docker Compose
cd /home/jenkins_data
docker compose up -d

# Using Docker command
docker start jenkins
```

#### Stopping Jenkins
```bash
# Using Docker Compose
cd /home/jenkins_data
docker compose down

# Using Docker command
docker stop jenkins
```

#### Restarting Jenkins
```bash
# Using Docker Compose
cd /home/jenkins_data
docker compose restart

# Using Docker command
docker restart jenkins
```

#### Checking Jenkins Status
```bash
# Container status
docker ps | grep jenkins

# Resource usage
docker stats jenkins --no-stream

# Logs
docker logs jenkins --tail 100 -f
```

### Monitoring Commands

#### System Health Check
```bash
#!/bin/bash
# Save as /home/jenkins_data/health_check.sh

echo "=== Jenkins Health Check ==="
echo "Date: $(date)"
echo

# Check container status
echo "Container Status:"
docker ps | grep jenkins || echo "Jenkins container not running!"
echo

# Check disk usage
echo "Disk Usage:"
df -h /home/jenkins_data
echo

# Check memory usage
echo "Memory Usage:"
docker stats jenkins --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"
echo

# Check if Jenkins is responsive
echo "Jenkins Responsiveness:"
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/login || echo "Jenkins not responding!"
echo
```

#### Log Monitoring
```bash
# Real-time logs
docker logs jenkins -f

# Error logs only
docker logs jenkins 2>&1 | grep -i error

# Last 24 hours logs
docker logs jenkins --since 24h
```

---

## Troubleshooting Guide

### Common Issues and Solutions

#### 1. Container Won't Start
**Symptom**: `docker compose up -d` fails or container exits immediately

**Diagnosis**:
```bash
# Check logs
docker logs jenkins

# Check if ports are in use
sudo netstat -tulpn | grep -E ":(8080|50000)"

# Check disk space
df -h /home
```

**Solutions**:
```bash
# If port conflict
docker compose down
# Edit docker-compose.yml and change ports to 8081:8080
docker compose up -d

# If disk space issue
sudo du -sh /home/jenkins_data/*
# Clean up old builds or increase disk space

# If permission issue
sudo chown -R 1000:1000 /home/jenkins_data/jenkins_home
```

#### 2. Jenkins Web Interface Not Accessible
**Symptom**: Cannot access http://VM_IP:8080

**Diagnosis**:
```bash
# Check if container is running
docker ps | grep jenkins

# Check if port is bound
sudo netstat -tulpn | grep :8080

# Check firewall
sudo ufw status
# or
sudo firewall-cmd --list-all
```

**Solutions**:
```bash
# Open firewall ports
sudo ufw allow 8080/tcp
sudo ufw allow 50000/tcp

# Or for CentOS/RHEL
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=50000/tcp
sudo firewall-cmd --reload
```

#### 3. Docker Socket Permission Issues
**Symptom**: Jenkins cannot execute Docker commands

**Diagnosis**:
```bash
# Check Docker socket permissions
ls -la /var/run/docker.sock

# Test from container
docker exec jenkins docker version
```

**Solutions**:
```bash
# Ensure container runs as root
# Already configured in our setup

# Or change socket permissions (less secure)
sudo chmod 666 /var/run/docker.sock
```

#### 4. Jenkins Running Out of Memory
**Symptom**: Java heap space errors, slow performance

**Diagnosis**:
```bash
# Check memory usage
docker stats jenkins --no-stream

# Check Java heap in logs
docker logs jenkins | grep -i "heap\|memory\|oom"
```

**Solutions**:
```bash
# Increase Java heap size
# Edit docker-compose.yml and change JAVA_OPTS
environment:
  - JAVA_OPTS=-Xmx4096m -Xms1024m

# Restart container
docker compose down
docker compose up -d
```

#### 5. Build Failures Due to Workspace Issues
**Symptom**: Builds failing with workspace-related errors

**Solutions**:
```bash
# Clean workspace plugin configuration
# Install "Workspace Cleanup Plugin"
# Add "Delete workspace before build starts" in job configuration

# Manual cleanup
docker exec jenkins find /var/jenkins_home/workspace -type f -name "*.tmp" -delete
```

### Emergency Procedures

#### Complete System Recovery
```bash
#!/bin/bash
# Emergency recovery script

echo "Starting Jenkins emergency recovery..."

# Stop Jenkins
docker stop jenkins 2>/dev/null

# Backup current state
sudo tar -czf /home/jenkins_backup_emergency_$(date +%Y%m%d_%H%M%S).tar.gz -C /home jenkins_data

# Remove problematic container
docker rm jenkins 2>/dev/null

# Restart with fresh container
cd /home/jenkins_data
docker compose up -d

echo "Recovery complete. Check logs: docker logs jenkins"
```

---

## Security Considerations

### Network Security
```bash
# Restrict access to specific IPs (example)
# Edit docker-compose.yml
ports:
  - "127.0.0.1:8080:8080"  # Only localhost
  - "192.168.1.100:8080:8080"  # Specific IP
```

### Authentication & Authorization
1. **Enable Security**:
   - Manage Jenkins → Configure Global Security
   - Enable "Jenkins' own user database"
   - Disable "Allow users to sign up"

2. **Matrix-based Security**:
   - Configure matrix-based security
   - Assign appropriate permissions to users/groups

### SSL/TLS Configuration
```bash
# Generate self-signed certificate
openssl req -new -x509 -days 365 -nodes -out /home/jenkins_data/jenkins.crt -keyout /home/jenkins_data/jenkins.key

# Update docker-compose.yml
environment:
  - JENKINS_OPTS=--httpPort=-1 --httpsPort=8443 --httpsKeyStore=/var/jenkins_home/jenkins.jks --httpsKeyStorePassword=password

volumes:
  - /home/jenkins_data/jenkins.crt:/var/jenkins_home/jenkins.crt
  - /home/jenkins_data/jenkins.key:/var/jenkins_home/jenkins.key
```

### Regular Security Updates
```bash
# Update Jenkins image
docker pull jenkins/jenkins:lts
docker compose down
docker compose up -d

# Update plugins
# Manage Jenkins → Manage Plugins → Updates
```

---

## Maintenance & Backup

### Backup Procedures

#### Daily Backup Script
```bash
#!/bin/bash
# Save as /home/jenkins_data/backup_daily.sh

BACKUP_DIR="/home/jenkins_backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="jenkins_backup_${DATE}.tar.gz"

# Create backup directory
mkdir -p $BACKUP_DIR

# Stop Jenkins for consistent backup
echo "Stopping Jenkins..."
cd /home/jenkins_data
docker compose down

# Create backup
echo "Creating backup..."
sudo tar -czf $BACKUP_DIR/$BACKUP_FILE -C /home jenkins_data

# Start Jenkins
echo "Starting Jenkins..."
docker compose up -d

# Remove backups older than 7 days
find $BACKUP_DIR -name "jenkins_backup_*.tar.gz" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/$BACKUP_FILE"
```

#### Weekly Full Backup
```bash
#!/bin/bash
# Save as /home/jenkins_data/backup_weekly.sh

BACKUP_DIR="/home/jenkins_backups/weekly"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Stop Jenkins
docker compose down

# Create comprehensive backup
sudo tar -czf $BACKUP_DIR/jenkins_full_backup_${DATE}.tar.gz \
  -C /home jenkins_data \
  --exclude='jenkins_data/jenkins_home/workspace/*/target' \
  --exclude='jenkins_data/jenkins_home/workspace/*/.git' \
  --exclude='jenkins_data/jenkins_home/logs/*'

# Start Jenkins
docker compose up -d

# Keep only last 4 weekly backups
ls -t $BACKUP_DIR/jenkins_full_backup_*.tar.gz | tail -n +5 | xargs -r rm

echo "Weekly backup completed"
```

### Restore Procedures

#### Complete System Restore
```bash
#!/bin/bash
# Restore from backup

BACKUP_FILE="$1"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file>"
    exit 1
fi

# Stop Jenkins
docker compose down

# Remove existing data
sudo rm -rf /home/jenkins_data.old
sudo mv /home/jenkins_data /home/jenkins_data.old

# Extract backup
sudo tar -xzf $BACKUP_FILE -C /home

# Fix permissions
sudo chown -R 1000:1000 /home/jenkins_data/jenkins_home

# Start Jenkins
cd /home/jenkins_data
docker compose up -d

echo "Restore completed from $BACKUP_FILE"
```

### System Updates

#### Update Jenkins
```bash
#!/bin/bash
# Update Jenkins to latest LTS

echo "Updating Jenkins..."

# Backup current state
sudo tar -czf /home/jenkins_backup_before_update_$(date +%Y%m%d_%H%M%S).tar.gz -C /home jenkins_data

# Pull latest image
docker pull jenkins/jenkins:lts

# Recreate container
cd /home/jenkins_data
docker compose down
docker compose up -d

echo "Update completed"
```

#### System Maintenance
```bash
#!/bin/bash
# Monthly maintenance script

echo "=== Jenkins Monthly Maintenance ==="

# Clean up old Docker images
docker image prune -f

# Clean up old logs
docker exec jenkins find /var/jenkins_home/logs -name "*.log" -mtime +30 -delete

# Clean up old workspaces
docker exec jenkins find /var/jenkins_home/workspace -name "target" -type d -mtime +7 -exec rm -rf {} +

# System disk cleanup
sudo journalctl --vacuum-time=30d

echo "Maintenance completed"
```

---

## Knowledge Transfer Checklist

### For New Team Members

#### Understanding the Setup
- [ ] Read and understand the architecture overview
- [ ] Understand why data is stored on `/home` instead of root
- [ ] Know the difference between Docker Compose and Docker run methods
- [ ] Understand the port mappings (8080, 50000)

#### Hands-on Tasks
- [ ] Successfully start and stop Jenkins
- [ ] Access Jenkins web interface
- [ ] Retrieve initial admin password
- [ ] Install a new plugin
- [ ] Check Jenkins logs
- [ ] Create a simple test job

#### Operational Knowledge
- [ ] Know how to check Jenkins status
- [ ] Understand how to read Docker logs
- [ ] Know where Jenkins data is stored
- [ ] Understand backup and restore procedures
- [ ] Know basic troubleshooting steps

#### Emergency Procedures
- [ ] Know how to restart Jenkins service
- [ ] Know how to recover from container failure
- [ ] Know how to restore from backup
- [ ] Know who to contact for major issues

### Key Information to Remember

#### Critical Paths
```bash
# Jenkins data directory
/home/jenkins_data/jenkins_home

# Docker compose file
/home/jenkins_data/docker-compose.yml

# Backup location
/home/jenkins_backups/

# Initial admin password
/home/jenkins_data/jenkins_home/secrets/initialAdminPassword
```

#### Important Commands
```bash
# Start/Stop Jenkins
docker compose up -d / docker compose down

# Check logs
docker logs jenkins

# Check status
docker ps | grep jenkins

# Backup
sudo tar -czf backup.tar.gz -C /home jenkins_data

# Restore
sudo tar -xzf backup.tar.gz -C /home
```

#### Contact Information
- **Primary Admin**: [Name] - [Email] - [Phone]
- **Secondary Admin**: [Name] - [Email] - [Phone]
- **Infrastructure Team**: [Email]
- **Emergency Contact**: [Phone]

---

## Appendix

### Useful Scripts

#### Health Check Script
```bash
#!/bin/bash
# /home/jenkins_data/scripts/health_check.sh

check_container() {
    if docker ps | grep -q jenkins; then
        echo "✓ Jenkins container is running"
        return 0
    else
        echo "✗ Jenkins container is not running"
        return 1
    fi
}

check_web_interface() {
    if curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/login | grep -q "200"; then
        echo "✓ Jenkins web interface is accessible"
        return 0
    else
        echo "✗ Jenkins web interface is not accessible"
        return 1
    fi
}

check_disk_space() {
    USAGE=$(df /home | awk 'NR==2 {print $5}' | sed 's/%//')
    if [ $USAGE -lt 80 ]; then
        echo "✓ Disk usage is acceptable ($USAGE%)"
        return 0
    else
        echo "⚠ Disk usage is high ($USAGE%)"
        return 1
    fi
}

echo "=== Jenkins Health Check ==="
echo "Date: $(date)"
echo

check_container
check_web_interface
check_disk_space

echo
echo "Health check completed"
```

#### Auto-restart Script
```bash
#!/bin/bash
# /home/jenkins_data/scripts/auto_restart.sh

MAX_RETRIES=3
RETRY_COUNT=0

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
    if docker ps | grep -q jenkins; then
        echo "Jenkins is running"
        exit 0
    else
        echo "Jenkins is not running, attempting to start..."
        cd /home/jenkins_data
        docker compose up -d
        sleep 30
        RETRY_COUNT=$((RETRY_COUNT + 1))
    fi
done

echo "Failed to start Jenkins after $MAX_RETRIES attempts"
exit 1
```

### Configuration Templates

#### Docker Compose for Different Environments

**Development Environment**
```yaml
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins-dev
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - /home/jenkins_data/jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JENKINS_OPTS=--httpPort=8080 --prefix=/jenkins
      - JAVA_OPTS=-Xmx1024m -Xms512m
    user: root
```

**Production Environment**
```yaml
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins-prod
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:50000:50000"
    volumes:
      - /home/jenkins_data/jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JENKINS_OPTS=--httpPort=8080
      - JAVA_OPTS=-Xmx4096m -Xms1024m
    user: root
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

## Document Revision History

| Version | Date | Changes | Author |
|---------|------|---------|---------|
| 1.0 | July 2025 | Initial documentation | Shubham kapadnis |

---

**End of Document**

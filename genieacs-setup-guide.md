# GenieACS Complete Setup Guide

A comprehensive guide to setting up GenieACS with a dockerized MongoDB database and systemd service management.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Install Node.js](#step-1-install-nodejs)
- [Step 2: Install Docker and Docker Compose](#step-2-install-docker-and-docker-compose)
- [Step 3: Setup MongoDB with Docker](#step-3-setup-mongodb-with-docker)
- [Step 4: Clone and Build GenieACS](#step-4-clone-and-build-genieacs)
- [Step 5: Configure GenieACS](#step-5-configure-genieacs)
- [Step 6: Create Startup Script](#step-6-create-startup-script)
- [Step 7: Create Systemd Service](#step-7-create-systemd-service)
- [Step 8: Start and Verify Services](#step-8-start-and-verify-services)
- [Troubleshooting](#troubleshooting)
- [Useful Commands](#useful-commands)

---

## Prerequisites

- Ubuntu/Debian Linux system (or similar)
- sudo privileges
- Basic command line knowledge

---

## Step 1: Install Node.js

GenieACS requires Node.js 12.3 or higher. We'll use NVM (Node Version Manager) for easy installation.

### Install NVM

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
```

### Load NVM into your current session

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

### Install Node.js 20.x

```bash
nvm install 20
nvm use 20
nvm alias default 20
```

### Verify installation

```bash
node --version
npm --version
```

You should see something like `v20.19.0` and `10.x.x`

### Find your Node.js path (save this for later)

```bash
which node
```

Example output: `/home/YOUR_USERNAME/.nvm/versions/node/v20.19.0/bin/node`

**⚠️ Important: Save this path! You'll need it for the systemd service.**

---

## Step 2: Install Docker and Docker Compose

### Install Docker

```bash
# Update package index
sudo apt-get update

# Install prerequisites
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### Add your user to docker group (to run docker without sudo)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Verify Docker installation

```bash
docker --version
docker compose version
```

---

## Step 3: Setup MongoDB with Docker

### Create a directory for your MongoDB setup

```bash
mkdir -p ~/mongodb
cd ~/mongodb
```

### Create docker-compose.yml file

```bash
nano docker-compose.yml
```

Paste the following content:

```yaml
services:
  mongodb:
    image: mongo:8.2.3@sha256:7f5bbdafebde7c42e42e33396d01c0eda3eb753da8dae99071a30e350568a0a4
    container_name: mongodb
    restart: unless-stopped
    ports:
      - "127.0.0.1:27027:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: admin
    volumes:
      - mongodb_data:/data/db
      - mongodb_config:/data/configdb

volumes:
  mongodb_data:
  mongodb_config:
```

Save and exit (Ctrl+X, then Y, then Enter)

### Start MongoDB container

```bash
docker compose up -d
```

### Verify MongoDB is running

```bash
docker ps
```

You should see the mongodb container running.

### Test MongoDB connection

First, install mongosh (MongoDB Shell):

```bash
# Install mongosh
wget -qO- https://www.mongodb.org/static/pgp/server-7.0.asc | sudo tee /etc/apt/trusted.gpg.d/mongodb.asc
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt-get update
sudo apt-get install -y mongodb-mongosh

# Test connection
mongosh "mongodb://admin:admin@localhost:27027/genieacs?authSource=admin"
```

If successful, you'll see the MongoDB shell. Type `exit` to quit.

### MongoDB Connection String

Your MongoDB connection string is:
```
mongodb://admin:admin@localhost:27027/genieacs?authSource=admin
```

**⚠️ Important: For production, change the default admin/admin credentials!**

---

## Step 4: Clone and Build GenieACS

### Clone GenieACS repository

```bash
cd ~
git clone https://github.com/genieacs/genieacs.git
cd genieacs
```

### Install dependencies and build

```bash
npm install
npm run build
```

This will create the `dist` directory with compiled binaries.

### Verify binaries exist

```bash
ls -la ~/genieacs/dist/bin/
```

You should see: `genieacs-cwmp`, `genieacs-nbi`, `genieacs-fs`, `genieacs-ui`

---

## Step 5: Configure GenieACS

Paste the following in `~/genieacs/lib/config.json`:

```json
MONGODB_CONNECTION_URL: {
  type: "string",
  default: "mongodb://admin:admin@localhost:27027/genieacs?authSource=admin",
}
```

Save and exit (Ctrl+X, then Y, then Enter)

### Test GenieACS manually (optional but recommended)

```bash
export GENIEACS_MONGODB_CONNECTION_URL="mongodb://admin:admin@localhost:27027/genieacs?authSource=admin"

cd ~/genieacs/dist/bin
./genieacs-cwmp &
./genieacs-nbi &
./genieacs-fs &
./genieacs-ui --ui-jwt-secret secret &
```

Check if all services are running:

```bash
ps aux | grep genieacs
```

Access the UI at http://localhost:3000

Kill the test processes:

```bash
pkill -f genieacs
```

---

## Step 6: Create Startup Script

### Get your Node.js path

```bash
which node
```

Example output: `/home/piva/.nvm/versions/node/v20.19.0/bin/node`

**⚠️ Copy this path - you'll need it in the next step!**

### Create the startup script

```bash
sudo nano /usr/local/bin/genieacs.sh
```

Paste the following (replace `/home/YOUR_USERNAME` and the Node path with your actual values):

```bash
#!/bin/bash

# Set environment variables
export GENIEACS_MONGODB_CONNECTION_URL="mongodb://admin:admin@localhost:27027/genieacs?authSource=admin"
export GENIEACS_UI_JWT_SECRET="secret"

# Set your Node.js path here (from 'which node')
NODE_PATH="/home/YOUR_USERNAME/.nvm/versions/node/v20.19.0/bin/node"

# Change to GenieACS bin directory
cd /home/YOUR_USERNAME/genieacs/dist/bin || exit 1

# Start services in background
$NODE_PATH genieacs-ui --ui-jwt-secret secret &
$NODE_PATH genieacs-fs &
$NODE_PATH genieacs-cwmp &
$NODE_PATH genieacs-nbi &

# Wait forever to keep the service alive
wait
```

**Important replacements:**
- Replace `/home/YOUR_USERNAME` with your actual home directory (e.g., `/home/piva`)
- Replace the `NODE_PATH` with your actual Node.js path from `which node`

Save and exit (Ctrl+X, then Y, then Enter)

### Make the script executable

```bash
sudo chmod +x /usr/local/bin/genieacs.sh
```

### Test the script manually

```bash
/usr/local/bin/genieacs.sh
```

Press Ctrl+C to stop after verifying it works.

---

## Step 7: Create Systemd Service

### Create the service file

```bash
sudo nano /etc/systemd/system/genieacs.service
```

Paste the following (replace `YOUR_USERNAME` and the Node.js bin path):

```ini
[Unit]
Description=GenieACS TR-069 Auto Configuration Server
After=network.target docker.service
Requires=docker.service

[Service]
Type=simple
ExecStart=/usr/local/bin/genieacs.sh
Restart=always
RestartSec=10
User=YOUR_USERNAME
Environment="GENIEACS_MONGODB_CONNECTION_URL=mongodb://admin:admin@localhost:27027/genieacs?authSource=admin"
Environment="PATH=/home/YOUR_USERNAME/.nvm/versions/node/v20.19.0/bin:/usr/local/bin:/usr/bin:/bin"
WorkingDirectory=/home/YOUR_USERNAME/genieacs/dist/bin
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Important replacements:**
- Replace `YOUR_USERNAME` with your actual username (e.g., `piva`)
- Replace the Node.js path in the `Environment="PATH=..."` line with your Node.js bin directory

Save and exit (Ctrl+X, then Y, then Enter)

### Reload systemd daemon

```bash
sudo systemctl daemon-reload
```

---

## Step 8: Start and Verify Services

### Enable GenieACS service to start on boot

```bash
sudo systemctl enable genieacs.service
```

### Start the GenieACS service

```bash
sudo systemctl start genieacs.service
```

### Check service status

```bash
sudo systemctl status genieacs.service
```

You should see "active (running)" in green.

### Check logs

```bash
sudo journalctl -u genieacs.service -f
```

Press Ctrl+C to stop viewing logs.

### Verify all services are listening on their ports

```bash
sudo lsof -i :3000  # UI
sudo lsof -i :7547  # CWMP
sudo lsof -i :7557  # NBI
sudo lsof -i :7567  # FS
```

Each should show a node process listening.

### Access GenieACS Web UI

Open your browser and navigate to:
```
http://localhost:3000
```

or if accessing remotely:
```
http://YOUR_SERVER_IP:3000
```

---

## Troubleshooting

### Service fails to start

**Check logs:**
```bash
sudo journalctl -u genieacs.service -n 50 --no-pager
```

**Common issues:**

1. **"node: command not found"**
   - The PATH environment variable in the service file is incorrect
   - Verify your Node.js path: `which node`
   - Update the PATH in `/etc/systemd/system/genieacs.service`

2. **"MongoDB connection refused"**
   - Check if MongoDB container is running: `docker ps`
   - Restart MongoDB: `cd ~/mongodb && docker compose restart`
   - Test connection: `mongosh "mongodb://admin:admin@localhost:27027/genieacs?authSource=admin"`

3. **"Permission denied"**
   - Make script executable: `sudo chmod +x /usr/local/bin/genieacs.sh`
   - Check file ownership: `ls -la /usr/local/bin/genieacs.sh`

### MongoDB not accessible

```bash
# Check if container is running
docker ps | grep mongodb

# Check container logs
docker logs mongodb

# Restart container
cd ~/mongodb
docker compose restart

# Check port binding
sudo lsof -i :27027
```

### GenieACS services not responding

```bash
# Restart the service
sudo systemctl restart genieacs.service

# Check if processes are running
ps aux | grep genieacs

# Check if ports are in use
sudo netstat -tulpn | grep -E ':(3000|7547|7557|7567)'
```

### View detailed GenieACS logs

```bash
# Follow logs in real-time
sudo journalctl -u genieacs.service -f

# View last 100 lines
sudo journalctl -u genieacs.service -n 100

# View logs since boot
sudo journalctl -u genieacs.service -b
```

---

## Useful Commands

### GenieACS Service Management

```bash
# Start service
sudo systemctl start genieacs.service

# Stop service
sudo systemctl stop genieacs.service

# Restart service
sudo systemctl restart genieacs.service

# Check status
sudo systemctl status genieacs.service

# Enable auto-start on boot
sudo systemctl enable genieacs.service

# Disable auto-start on boot
sudo systemctl disable genieacs.service

# View logs
sudo journalctl -u genieacs.service -f
```

### MongoDB Docker Management

```bash
# Start MongoDB
cd ~/mongodb && docker compose up -d

# Stop MongoDB
cd ~/mongodb && docker compose down

# Restart MongoDB
cd ~/mongodb && docker compose restart

# View MongoDB logs
docker logs mongodb -f

# Stop and remove everything (including data)
cd ~/mongodb && docker compose down -v
```

### Check Service Ports

```bash
# Check all GenieACS ports
sudo lsof -i :3000   # UI
sudo lsof -i :7547   # CWMP
sudo lsof -i :7557   # NBI
sudo lsof -i :7567   # FS
sudo lsof -i :27027  # MongoDB

# Alternative using netstat
sudo netstat -tulpn | grep -E ':(3000|7547|7557|7567|27027)'
```

### Update GenieACS

```bash
# Stop service
sudo systemctl stop genieacs.service

# Pull latest code
cd ~/genieacs
git pull

# Rebuild
npm install
npm run build

# Start service
sudo systemctl start genieacs.service
```

---

## Network Configuration

### Port Requirements

| Service | Port | Description |
|---------|------|-------------|
| GenieACS UI | 3000 | Web interface |
| GenieACS CWMP | 7547 | TR-069 ACS (CPE connects here) |
| GenieACS NBI | 7557 | REST API (Northbound Interface) |
| GenieACS FS | 7567 | File Server (firmware downloads) |
| MongoDB | 27027 | Database (internal only) |

### Firewall Configuration (if needed)

```bash
# Allow GenieACS ports
sudo ufw allow 3000/tcp   # UI
sudo ufw allow 7547/tcp   # CWMP
sudo ufw allow 7557/tcp   # NBI (optional, only if using API)
sudo ufw allow 7567/tcp   # FS (optional, only if serving files)

# MongoDB should NOT be exposed externally
# It's only accessible on localhost (127.0.0.1)
```

---

## Production Recommendations

### Security Hardening

1. **Change MongoDB credentials:**
   ```bash
   # Edit docker-compose.yml
   nano ~/mongodb/docker-compose.yml
   # Change MONGO_INITDB_ROOT_USERNAME and MONGO_INITDB_ROOT_PASSWORD
   
   # Restart MongoDB
   cd ~/mongodb && docker compose down && docker compose up -d
   
   # Update GenieACS config
   nano ~/genieacs/config/config.json
   # Update connection string with new credentials
   
   # Update systemd service
   sudo nano /etc/systemd/system/genieacs.service
   # Update GENIEACS_MONGODB_CONNECTION_URL
   
   # Update startup script
   sudo nano /usr/local/bin/genieacs.sh
   # Update connection string
   
   # Restart GenieACS
   sudo systemctl restart genieacs.service
   ```

2. **Use SSL/TLS for GenieACS services** (for production)

3. **Change UI_JWT_SECRET** to a strong random string

4. **Regular backups of MongoDB data:**
   ```bash
   # Backup MongoDB
   docker exec mongodb mongodump --authenticationDatabase admin -u admin -p admin --out /backup
   docker cp mongodb:/backup ./mongodb-backup-$(date +%Y%m%d)
   ```

5. **Enable authentication on GenieACS UI** (configure in the UI itself)

### Monitoring

Check service health regularly:
```bash
# Create a health check script
cat > ~/check-genieacs.sh << 'EOF'
#!/bin/bash
echo "=== GenieACS Service Status ==="
systemctl is-active genieacs.service

echo -e "\n=== MongoDB Container Status ==="
docker ps --filter name=mongodb --format "{{.Status}}"

echo -e "\n=== Port Status ==="
for port in 3000 7547 7557 7567 27027; do
  if sudo lsof -i :$port > /dev/null 2>&1; then
    echo "Port $port: OPEN"
  else
    echo "Port $port: CLOSED"
  fi
done
EOF

chmod +x ~/check-genieacs.sh
```

Run the health check:
```bash
~/check-genieacs.sh
```

---

## Backup and Restore

### Backup MongoDB Data

```bash
# Create backup directory
mkdir -p ~/backups

# Backup MongoDB
docker exec mongodb mongodump \
  --authenticationDatabase admin \
  -u admin \
  -p admin \
  --out /backup

# Copy backup from container
docker cp mongodb:/backup ~/backups/mongodb-backup-$(date +%Y%m%d)

# Create compressed archive
cd ~/backups
tar -czf mongodb-backup-$(date +%Y%m%d).tar.gz mongodb-backup-$(date +%Y%m%d)
```

### Restore MongoDB Data

```bash
# Extract backup
cd ~/backups
tar -xzf mongodb-backup-YYYYMMDD.tar.gz

# Copy to container
docker cp mongodb-backup-YYYYMMDD mongodb:/restore

# Restore
docker exec mongodb mongorestore \
  --authenticationDatabase admin \
  -u admin \
  -p admin \
  /restore/mongodb-backup-YYYYMMDD
```

---

## Additional Resources

- **GenieACS Documentation:** https://docs.genieacs.com
- **GenieACS GitHub:** https://github.com/genieacs/genieacs
- **GenieACS Forum:** https://forum.genieacs.com
- **MongoDB Documentation:** https://docs.mongodb.com
- **Docker Documentation:** https://docs.docker.com

---

## Summary of File Locations

```
Configuration Files:
├── ~/mongodb/docker-compose.yml           # MongoDB Docker configuration
├── ~/genieacs/config/config.json          # GenieACS configuration
├── /usr/local/bin/genieacs.sh             # GenieACS startup script
└── /etc/systemd/system/genieacs.service   # Systemd service file

GenieACS Installation:
└── ~/genieacs/
    ├── dist/bin/                          # Compiled binaries
    ├── config/                            # Configuration directory
    └── node_modules/                      # Dependencies

MongoDB Data (Docker volumes):
├── mongodb_data                           # Database files
└── mongodb_config                         # MongoDB configuration
```

---

## Quick Start Checklist

- [ ] Install Node.js via NVM
- [ ] Install Docker and Docker Compose
- [ ] Create MongoDB docker-compose.yml
- [ ] Start MongoDB container
- [ ] Clone GenieACS repository
- [ ] Build GenieACS
- [ ] Create GenieACS config.json
- [ ] Find Node.js path with `which node`
- [ ] Create genieacs.sh script with correct paths
- [ ] Make script executable
- [ ] Create systemd service file with correct paths
- [ ] Reload systemd daemon
- [ ] Enable and start GenieACS service
- [ ] Verify all services are running
- [ ] Access UI at http://localhost:3000

---

**Setup Complete! 🎉**

Your GenieACS installation is now ready. You can access the web interface at http://localhost:3000 and start managing your TR-069 devices.
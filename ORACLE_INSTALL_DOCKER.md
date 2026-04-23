# Oracle Database Installation via Docker

This guide shows how to install Oracle Database using official Oracle container images on a Linux VM (Ubuntu-focused). This is the easiest way to get Oracle running for CDC testing.

## Prerequisites

- Linux VM (Ubuntu 20.04+, Debian, or similar - also works on RHEL/CentOS)
- Root or sudo access
- At least 8GB RAM (4GB minimum for XE, more recommended)
- 20GB+ free disk space
- Internet connection to pull Docker images

## Option 1: Quick Install Script (Ubuntu / Debian)

Save this as `install_oracle_docker.sh`:

```bash
#!/bin/bash
# Oracle Database Docker Installation Script
# For Ubuntu / Debian

set -e

# Configuration - EDIT THESE
ORACLE_VERSION="21c"          # Options: 21c, 23ai, 26ai
ORACLE_PASSWORD="confluent123"
ORACLE_MEMORY="4000"          # MB of memory for Oracle

echo "=== Installing Oracle ${ORACLE_VERSION} via Docker ==="

# Step 1: Install Docker
echo "[1/5] Installing Docker..."
sudo apt-get update
sudo apt-get install -y docker.io wget unzip jq curl

# Configure Docker
echo "[2/5] Configuring Docker..."
sudo usermod -a -G docker $USER
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -w vm.max_map_count=262144
echo "    *       soft  nofile  65535
    *       hard  nofile  65535" | sudo tee -a /etc/security/limits.conf

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker

# Step 3: Download setup scripts
echo "[3/5] Downloading Oracle setup scripts..."
cd $HOME
wget -O cdc-scripts.zip https://github.com/ora0600/confluent-new-cdc-connector/archive/refs/heads/main.zip
unzip -q cdc-scripts.zip
mkdir -p oracle-docker

# Copy scripts based on version
if [ "$ORACLE_VERSION" = "21c" ]; then
    cp -r confluent-new-cdc-connector-main/oraclexe21c/docker/scripts oracle-docker/
elif [ "$ORACLE_VERSION" = "23ai" ]; then
    cp -r confluent-new-cdc-connector-main/oracle23ai/docker/scripts oracle-docker/
elif [ "$ORACLE_VERSION" = "26ai" ]; then
    cp -r confluent-new-cdc-connector-main/oracle26ai/docker/scripts oracle-docker/
else
    echo "Invalid Oracle version: $ORACLE_VERSION"
    exit 1
fi

rm -rf confluent-new-cdc-connector-main cdc-scripts.zip

# Step 4: Run Oracle container
echo "[4/5] Starting Oracle container (this will take several minutes)..."
cd oracle-docker

if [ "$ORACLE_VERSION" = "21c" ]; then
    sudo docker run --name oracle21c \
        -p 1521:1521 -p 5500:5500 -p 8080:8080 \
        -e ORACLE_SID=XE \
        -e ORACLE_PDB=XEPDB1 \
        -e ORACLE_PWD=$ORACLE_PASSWORD \
        -e ORACLE_MEM=$ORACLE_MEMORY \
        -e ORACLE_CHARACTERSET=AL32UTF8 \
        -e ENABLE_ARCHIVELOG=true \
        -v /opt/oracle/oradata \
        -v $HOME/oracle-docker/scripts:/opt/oracle/scripts/setup \
        -d container-registry.oracle.com/database/express:21.3.0-xe
    
    CONTAINER_NAME="oracle21c"
    
elif [ "$ORACLE_VERSION" = "23ai" ]; then
    sudo docker run --name oracle23ai \
        -p 1521:1521 \
        -e ORACLE_PWD=$ORACLE_PASSWORD \
        -e ORACLE_MEM=$ORACLE_MEMORY \
        -e ORACLE_CHARACTERSET=AL32UTF8 \
        -e ENABLE_ARCHIVELOG=true \
        -e ENABLE_FORCE_LOGGING=true \
        -v /opt/oracle/oradata \
        -v $HOME/oracle-docker/scripts:/opt/oracle/scripts/setup \
        -d container-registry.oracle.com/database/free:23.5.0.0
    
    CONTAINER_NAME="oracle23ai"
    
elif [ "$ORACLE_VERSION" = "26ai" ]; then
    sudo docker run --name oracle26ai \
        -p 1521:1521 \
        -e ORACLE_PWD=$ORACLE_PASSWORD \
        -e ORACLE_MEM=$ORACLE_MEMORY \
        -e ORACLE_CHARACTERSET=AL32UTF8 \
        -e ENABLE_ARCHIVELOG=true \
        -e ENABLE_FORCE_LOGGING=true \
        -v /opt/oracle/oradata \
        -v $HOME/oracle-docker/scripts:/opt/oracle/scripts/setup \
        -d container-registry.oracle.com/database/free:latest
    
    CONTAINER_NAME="oracle26ai"
fi

# Step 5: Wait for Oracle to start and configure for CDC
echo "[5/5] Waiting for Oracle to start (this takes 5-10 minutes)..."
echo "You can monitor progress with: sudo docker logs -f $CONTAINER_NAME"
sleep 120

# Check if database is ready
echo "Checking if database is ready..."
for i in {1..30}; do
    if sudo docker exec $CONTAINER_NAME bash -c "echo 'SELECT 1 FROM DUAL;' | sqlplus -s sys/$ORACLE_PASSWORD@XE as sysdba" 2>/dev/null | grep -q "1"; then
        echo "Database is ready!"
        break
    fi
    echo "Still waiting... ($i/30)"
    sleep 20
done

# Run CDC setup
echo "Configuring database for CDC..."
sudo docker exec $CONTAINER_NAME /bin/bash -c "bash /opt/oracle/scripts/setup/00_setup_cdc.sh"

echo ""
echo "=== Installation Complete! ==="
echo ""
echo "Oracle $ORACLE_VERSION is running in container: $CONTAINER_NAME"
echo ""
echo "Connection Details:"
if [ "$ORACLE_VERSION" = "21c" ]; then
    echo "  CDB: XE"
    echo "  PDB: XEPDB1"
    echo "  Port: 1521"
    echo "  SYS password: $ORACLE_PASSWORD"
    echo ""
    echo "Connect with:"
    echo "  docker exec -it $CONTAINER_NAME sqlplus sys/$ORACLE_PASSWORD@XE as sysdba"
    echo "  docker exec -it $CONTAINER_NAME sqlplus sys/$ORACLE_PASSWORD@XEPDB1 as sysdba"
    echo "  docker exec -it $CONTAINER_NAME sqlplus ordermgmt/kafka@XEPDB1"
else
    echo "  CDB: FREE"
    echo "  PDB: FREEPDB1"
    echo "  Port: 1521"
    echo "  SYS password: $ORACLE_PASSWORD"
    echo ""
    echo "Connect with:"
    echo "  docker exec -it $CONTAINER_NAME sqlplus sys/$ORACLE_PASSWORD@FREE as sysdba"
    echo "  docker exec -it $CONTAINER_NAME sqlplus sys/$ORACLE_PASSWORD@FREEPDB1 as sysdba"
    echo "  docker exec -it $CONTAINER_NAME sqlplus ordermgmt/kafka@FREEPDB1"
fi
echo ""
echo "Users created:"
echo "  - ordermgmt/kafka (application data)"
echo "  - c##ggadmin/Confluent12! (XStream CDC)"
echo ""
echo "Next steps:"
echo "  1. Create XStream Outbound Server (see XSTREAM_SETUP.md)"
echo "  2. Configure Confluent CDC Connector"
echo ""
echo "Container commands:"
echo "  docker logs $CONTAINER_NAME             # View logs"
echo "  docker exec -it $CONTAINER_NAME bash    # Enter container"
echo "  docker stop $CONTAINER_NAME             # Stop database"
echo "  docker start $CONTAINER_NAME            # Start database"
echo "  docker rm $CONTAINER_NAME               # Remove container"
echo ""
echo "IMPORTANT: Log out and back in for Docker group membership to take effect!"
echo "Or run: newgrp docker"
echo ""
```

Make it executable and run:

```bash
chmod +x install_oracle_docker.sh
./install_oracle_docker.sh

# IMPORTANT: After installation, either logout/login or run:
newgrp docker

# Then verify Docker works without sudo:
docker ps
```

## Option 2: Manual Installation (Step by Step)

### Step 1: Install Docker

**On Ubuntu / Debian (using docker.io - recommended for simplicity):**

```bash
sudo apt-get update
sudo apt-get install -y docker.io wget unzip jq

# Configure Docker
sudo usermod -a -G docker $USER
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -w vm.max_map_count=262144

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker

# Activate Docker group (choose one):
newgrp docker          # Start new shell with docker group
# OR logout and login again
```

**On RHEL / CentOS / Amazon Linux 2:**
```bash
sudo yum update -y
sudo yum install -y docker wget unzip jq

# Configure system for Docker
sudo usermod -a -G docker $USER
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -w vm.max_map_count=262144

# Start Docker
sudo service docker start
sudo chkconfig docker on

# Log out and back in for group membership to take effect
```

### Step 2: Prepare Setup Scripts

Download the CDC setup scripts:

```bash
cd $HOME
wget https://github.com/ora0600/confluent-new-cdc-connector/archive/refs/heads/main.zip
unzip main.zip
mkdir -p oracle-docker

# For Oracle 21c XE:
cp -r confluent-new-cdc-connector-main/oraclexe21c/docker/scripts oracle-docker/

# For Oracle 23ai:
cp -r confluent-new-cdc-connector-main/oracle23ai/docker/scripts oracle-docker/

# For Oracle 26ai:
cp -r confluent-new-cdc-connector-main/oracle26ai/docker/scripts oracle-docker/

cd oracle-docker
```

### Step 3: Run Oracle Container

**Oracle 21c XE (Express Edition):**
```bash
# Make sure you're in the oracle-docker directory with the scripts folder
cd $HOME/oracle-docker

docker run --name oracle21c \
    -p 1521:1521 -p 5500:5500 -p 8080:8080 \
    -e ORACLE_SID=XE \
    -e ORACLE_PDB=XEPDB1 \
    -e ORACLE_PWD=confluent123 \
    -e ORACLE_MEM=4000 \
    -e ORACLE_CHARACTERSET=AL32UTF8 \
    -e ENABLE_ARCHIVELOG=true \
    -v /opt/oracle/oradata \
    -v $PWD/scripts:/opt/oracle/scripts/setup \
    -d container-registry.oracle.com/database/express:21.3.0-xe

# CDB: XE, PDB: XEPDB1
```

**Oracle 23ai FREE:**
```bash
docker run --name oracle23ai \
    -p 1521:1521 \
    -e ORACLE_PWD=confluent123 \
    -e ORACLE_MEM=4000 \
    -e ORACLE_CHARACTERSET=AL32UTF8 \
    -e ENABLE_ARCHIVELOG=true \
    -e ENABLE_FORCE_LOGGING=true \
    -v /opt/oracle/oradata \
    -v $PWD/scripts:/opt/oracle/scripts/setup \
    -d container-registry.oracle.com/database/free:23.5.0.0

# CDB: FREE, PDB: FREEPDB1
```

**Oracle 26ai FREE (Latest):**
```bash
docker run --name oracle26ai \
    -p 1521:1521 \
    -e ORACLE_PWD=confluent123 \
    -e ORACLE_MEM=4000 \
    -e ORACLE_CHARACTERSET=AL32UTF8 \
    -e ENABLE_ARCHIVELOG=true \
    -e ENABLE_FORCE_LOGGING=true \
    -v /opt/oracle/oradata \
    -v $PWD/scripts:/opt/oracle/scripts/setup \
    -d container-registry.oracle.com/database/free:latest

# CDB: FREE, PDB: FREEPDB1
```

### Step 4: Wait for Oracle to Start

Oracle takes 5-10 minutes to initialize on first run. Monitor progress:

```bash
# Watch the logs
docker logs -f oracle21c   # or oracle23ai, oracle26ai

# You'll see messages like:
# DATABASE IS READY TO USE!
```

Or check status:
```bash
# Keep checking until it returns data
docker exec oracle21c bash -c \
    "echo 'SELECT 1 FROM DUAL;' | sqlplus -s sys/confluent123@XE as sysdba"
```

### Step 5: Configure for CDC

Once Oracle is ready, run the CDC setup:

```bash
# For Oracle 21c XE:
docker exec oracle21c /bin/bash -c "bash /opt/oracle/scripts/setup/00_setup_cdc.sh"

# For Oracle 23ai:
docker exec oracle23ai /bin/bash -c "bash /opt/oracle/scripts/setup/00_setup_cdc.sh"

# For Oracle 26ai:
docker exec oracle26ai /bin/bash -c "bash /opt/oracle/scripts/setup/00_setup_cdc.sh"
```

This script will:
1. Enable archivelog mode
2. Configure redo logs (2GB each)
3. Enable supplemental logging
4. Create users (ordermgmt, c##ggadmin, c##cfltuser)
5. Create schema and load sample data
6. Create data generator procedures

## Verification

Test your installation:

```bash
# Enter the container
docker exec -it oracle21c bash   # or oracle23ai, oracle26ai

# Connect to Oracle
sqlplus sys/confluent123@XE as sysdba   # For 21c XE
# OR
sqlplus sys/confluent123@FREE as sysdba  # For 23ai/26ai

# Check status
SQL> SELECT instance_name, status FROM v$instance;
-- Should show: STATUS = OPEN

# Check users
SQL> SELECT username FROM dba_users WHERE username IN ('ORDERMGMT', 'C##GGADMIN');

# Check tables
SQL> SELECT table_name FROM dba_tables WHERE owner = 'ORDERMGMT';
-- Should show: ORDERS, CUSTOMERS, PRODUCTS, etc.

# Test application user
SQL> CONNECT ordermgmt/kafka@XEPDB1   -- For 21c
-- OR
SQL> CONNECT ordermgmt/kafka@FREEPDB1  -- For 23ai/26ai

SQL> SELECT COUNT(*) FROM orders;

SQL> EXIT
```

## Container Management

### Common Commands

```bash
# View logs
docker logs oracle21c
docker logs -f oracle21c  # Follow logs

# Stop database
docker stop oracle21c

# Start database
docker start oracle21c

# Restart database
docker restart oracle21c

# Enter container shell
docker exec -it oracle21c bash

# Check container status
docker ps -a | grep oracle

# Remove container (WARNING: deletes all data)
docker rm -f oracle21c
```

### Persistence

The `/opt/oracle/oradata` volume persists your data. To completely remove Oracle:

```bash
# Stop and remove container
docker stop oracle21c
docker rm oracle21c

# Find and remove the data volume
docker volume ls
docker volume rm <volume_id>
```

### Auto-start on Boot

**Using systemd (Ubuntu/RHEL 8+):**

Create `/etc/systemd/system/oracle21c.service`:

```ini
[Unit]
Description=Oracle 21c XE Container
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker start -a oracle21c
ExecStop=/usr/bin/docker stop oracle21c

[Install]
WantedBy=multi-user.target
```

Enable:
```bash
sudo systemctl daemon-reload
sudo systemctl enable oracle21c
sudo systemctl start oracle21c
```

## Connecting from External Tools

### SQL Developer / DBeaver

- **Host:** `<your-vm-ip>`
- **Port:** `1521`
- **Service Name:** `XE` (for 21c) or `FREE` (for 23ai/26ai)
- **Username:** `sys`
- **Password:** `confluent123`
- **Role:** `SYSDBA`

For application user:
- **Service Name:** `XEPDB1` (for 21c) or `FREEPDB1` (for 23ai/26ai)
- **Username:** `ordermgmt`
- **Password:** `kafka`

### JDBC Connection String

```
# Oracle 21c XE
jdbc:oracle:thin:@//<host>:1521/XEPDB1

# Oracle 23ai/26ai
jdbc:oracle:thin:@//<host>:1521/FREEPDB1
```

## Troubleshooting

### Container won't start
```bash
# Check Docker logs
docker logs oracle21c

# Check system resources
free -h
df -h

# Ensure Docker is running
sudo service docker status
```

### Oracle startup fails
```bash
# Check memory allocation
# XE needs minimum 2GB, recommended 4GB
docker inspect oracle21c | grep -i memory

# Check disk space
df -h

# View Oracle alert log
docker exec oracle21c tail -100 /opt/oracle/diag/rdbms/xe/XE/trace/alert_XE.log
```

### Cannot connect to Oracle
```bash
# Check if port 1521 is exposed
docker port oracle21c

# Check if Oracle listener is running
docker exec oracle21c bash -c "lsnrctl status"

# Check firewall (Ubuntu)
sudo ufw status
sudo ufw allow 1521/tcp

# For RHEL/CentOS:
# sudo firewall-cmd --list-ports
# sudo firewall-cmd --add-port=1521/tcp --permanent
# sudo firewall-cmd --reload
```

### "Database not ready" after 10 minutes
```bash
# Check if initialization is still running
docker exec oracle21c ps aux | grep oracle

# Check alert log for errors
docker exec oracle21c tail -200 /opt/oracle/diag/rdbms/xe/XE/trace/alert_XE.log

# For FREE edition, alert log is at:
docker exec oracle23ai tail -200 /opt/oracle/diag/rdbms/free/FREE/trace/alert_FREE.log
```

## Image Details

Oracle provides these official container images:

- **XE 21c:** Free, limited to 2GB RAM, 12GB user data
- **FREE 23ai:** Free, limited to 2GB RAM, 12GB user data, includes latest features
- **FREE 26ai:** Free, latest version with AI features

All images include:
- Oracle Database
- Oracle Enterprise Manager Express (port 5500 for XE, varies for FREE)
- SQL*Plus
- Oracle Net Listener

## Next Steps

After Oracle is installed and running:

1. **Create XStream Outbound Server** - See `XSTREAM_SETUP.md`
2. **Install Confluent Connector** - Point it to your Oracle database
3. **Start generating test data:**
   ```sql
   sqlplus ordermgmt/kafka@XEPDB1
   SQL> BEGIN produce_orders; END;
   ```

## Alternative: Using docker-compose

If you prefer docker-compose, create `docker-compose.yml`:

```yaml
version: '3.8'
services:
  oracle21c:
    container_name: oracle21c
    image: container-registry.oracle.com/database/express:21.3.0-xe
    ports:
      - "1521:1521"
      - "5500:5500"
      - "8080:8080"
    environment:
      - ORACLE_SID=XE
      - ORACLE_PDB=XEPDB1
      - ORACLE_PWD=confluent123
      - ORACLE_MEM=4000
      - ENABLE_ARCHIVELOG=true
      - ORACLE_CHARACTERSET=AL32UTF8
    volumes:
      - oracle-data:/opt/oracle/oradata
      - ./scripts:/opt/oracle/scripts/setup

volumes:
  oracle-data:
```

Run with:
```bash
docker-compose up -d
```

## Resources

- Oracle Container Registry: https://container-registry.oracle.com
- Oracle Database Free: https://www.oracle.com/database/free/
- Oracle Database XE Documentation: https://docs.oracle.com/en/database/oracle/oracle-database/21/xeinl/

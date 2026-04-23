# Quick Start: Oracle CDC on Linux VM (Standalone)

This guide shows you how to set up Oracle Database with XStream CDC on a Ubuntu/Debian Linux VM from scratch, without AWS/Terraform. Also works on RHEL/CentOS with minor adjustments.

## 3-Step Process

### Step 1: Install Oracle Database
**Time: 10-15 minutes**

Follow **[ORACLE_INSTALL_DOCKER.md](ORACLE_INSTALL_DOCKER.md)** to:
- Install Docker on your Linux VM
- Pull and run official Oracle container images
- Choose from Oracle 21c XE, 23ai FREE, or 26ai FREE

**Quick install:**
```bash
wget https://raw.githubusercontent.com/ora0600/confluent-new-cdc-connector/main/ORACLE_INSTALL_DOCKER.md
# Follow the Quick Install Script section
```

Result: Oracle Database running in Docker with initial configuration.

### Step 2: Configure for CDC (Optional - auto-done if using install script)
**Time: 5 minutes**

If you didn't use the install script, follow **[ORACLE_SETUP_STANDALONE.md](ORACLE_SETUP_STANDALONE.md)** to:
- Enable archivelog mode and supplemental logging
- Configure redo logs for CDC
- Create users (ordermgmt, c##ggadmin)
- Create schema with sample data

**Note:** The install script from Step 1 automatically runs these configurations. Only needed if installing Oracle manually.

### Step 3: Create XStream Outbound Server
**Time: 2 minutes**

Follow **[XSTREAM_SETUP.md](XSTREAM_SETUP.md)** to:
- Create the XStream outbound server
- Configure capture process
- Verify it's working

**Quick setup:**
```bash
sqlplus c##ggadmin/Confluent12!@XE   # For 21c
# Run the CREATE_OUTBOUND procedure from XSTREAM_SETUP.md
```

Result: Oracle ready for Confluent CDC Connector to attach.

## What You Get

After completing all 3 steps:

✅ Oracle Database running in Docker  
✅ Archivelog and supplemental logging enabled  
✅ Sample schema with orders, customers, products (17 tables)  
✅ XStream outbound server configured  
✅ Test data generator procedures  
✅ CDC users with proper permissions  

**Connection Details:**
- **Oracle 21c XE:** `localhost:1521/XEPDB1`
- **Oracle 23ai/26ai:** `localhost:1521/FREEPDB1`
- **SYS password:** `confluent123`
- **App user:** `ordermgmt/kafka`
- **CDC user:** `c##ggadmin/Confluent12!`

## Optional: Install Oracle Client on macOS

If you're on a Mac and want to run `sqlplus` directly from your terminal (instead of using `docker exec`), see:

📄 **[ORACLE_CLIENT_MACOS_SETUP.md](ORACLE_CLIENT_MACOS_SETUP.md)** - Install Oracle Instant Client on macOS

This allows you to run commands like:
```bash
sqlplus ordermgmt/kafka@localhost:1521/XEPDB1
sqlplus c##ggadmin/Confluent12!@localhost:1521/XE
```

Without needing to exec into the Docker container first.

## Next: Connect Confluent CDC Connector

Configure your Confluent XStream CDC Connector with these settings:

```json
{
  "connector.class": "io.confluent.connect.oracle.xstream.cdc.OracleXStreamSourceConnector",
  "database.hostname": "YOUR_VM_IP",
  "database.port": 1521,
  "database.user": "C##GGADMIN",
  "database.password": "Confluent12!",
  "database.dbname": "XE",                    // or "FREE" for 23ai/26ai
  "database.service.name": "XE",              // or "FREE" for 23ai/26ai
  "database.pdb.name": "XEPDB1",              // or "FREEPDB1" for 23ai/26ai
  "database.out.server.name": "XOUT",
  "table.include.list": "ORDERMGMT[.](ORDERS|ORDER_ITEMS|CUSTOMERS|PRODUCTS|EMPLOYEES)",
  "topic.prefix": "XEPDB1"                    // or "FREEPDB1" for 23ai/26ai
}
```

## Testing Your Setup

Generate test data:
```bash
docker exec -it oracle21c sqlplus ordermgmt/kafka@XEPDB1

SQL> BEGIN
  produce_orders;  -- Generates orders every 5 seconds for 5 hours
END;
/
```

Monitor XStream:
```bash
docker exec -it oracle21c sqlplus c##ggadmin/Confluent12!@XE

SQL> -- Check capture status
SELECT CAPTURE_NAME, STATE FROM V$XSTREAM_CAPTURE;

SQL> -- Check latency (should be < 10 seconds)
SELECT CAPTURE_NAME,
       ((SYSDATE - CAPTURE_MESSAGE_CREATE_TIME)*86400) LATENCY_SECONDS
FROM V$XSTREAM_CAPTURE;

SQL> -- Check messages sent
SELECT SERVER_NAME,
       TOTAL_TRANSACTIONS_SENT,
       TOTAL_MESSAGES_SENT,
       (BYTES_SENT/1024)/1024 as MB_SENT
FROM V$XSTREAM_OUTBOUND_SERVER;
```

## Estimated Times

- **Total setup time:** 20-25 minutes
- **Oracle container first start:** 5-10 minutes
- **Configuration scripts:** 5 minutes
- **XStream setup:** 2 minutes

## System Requirements

**Minimum:**
- 2 CPU cores
- 8GB RAM (4GB for Oracle, 2GB for OS, 2GB buffer)
- 20GB disk space
- Ubuntu 20.04+ or Debian 11+ (also works on RHEL/CentOS)

**Recommended:**
- 4+ CPU cores
- 16GB+ RAM
- 50GB+ SSD storage
- Ubuntu 22.04 LTS or 24.04 LTS
- Gigabit network connection

## Troubleshooting

**Docker permission denied:**
```bash
# After adding user to docker group, activate it:
newgrp docker
# Or logout and login again

# Test:
docker ps  # Should work without sudo
```

**Oracle won't start:**
```bash
# Check logs
docker logs oracle21c

# Common issue: not enough memory
# Reduce ORACLE_MEM in docker run command to 2000 (2GB)

# Check available memory:
free -h
```

**Can't connect from remote machine:**
```bash
# Ubuntu firewall:
sudo ufw status
sudo ufw allow 1521/tcp
sudo ufw reload

# Check Oracle listener is running:
docker exec oracle21c bash -c "lsnrctl status"

# Check if port is accessible:
netstat -tlnp | grep 1521
```

**XStream not capturing changes:**
```bash
# Verify supplemental logging
sqlplus sys/confluent123@XE as sysdba
SQL> SELECT supplemental_log_data_all FROM v$database;
-- Should return YES

# Check capture state
SQL> CONNECT c##ggadmin/Confluent12!@XE
SQL> SELECT CAPTURE_NAME, STATE FROM V$XSTREAM_CAPTURE;
-- Should show CAPTURING CHANGES or WAITING FOR REDO
```

## Complete Example: Ubuntu 22.04 / 24.04

**Quick automated install:**
```bash
# Download the install script
wget https://raw.githubusercontent.com/ora0600/confluent-new-cdc-connector/main/ORACLE_INSTALL_DOCKER.md
# Extract and save the install_oracle_docker.sh script from the document

# Or use this one-liner to create the script:
curl -o install_oracle_docker.sh https://your-repo/install_oracle_docker.sh
chmod +x install_oracle_docker.sh

# Edit the script to set your Oracle version (21c, 23ai, or 26ai)
# Then run it:
./install_oracle_docker.sh

# After script completes, activate Docker group:
newgrp docker
```

**Manual step-by-step (if you prefer to see each step):**
```bash
# 1. Install Docker
sudo apt-get update
sudo apt-get install -y docker.io wget unzip jq
sudo usermod -a -G docker $USER
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -w vm.max_map_count=262144
sudo systemctl start docker
sudo systemctl enable docker

# Activate Docker group (choose one):
newgrp docker           # Start new shell with docker group
# OR logout and login again

# 2. Download Oracle setup scripts
cd $HOME
mkdir -p oracle-docker
cd oracle-docker
wget https://github.com/ora0600/confluent-new-cdc-connector/archive/refs/heads/main.zip
unzip main.zip

# For Oracle 23ai (recommended):
cp -r confluent-new-cdc-connector-main/oracle23ai/docker/scripts .

# For Oracle 21c XE:
# cp -r confluent-new-cdc-connector-main/oraclexe21c/docker/scripts .

# For Oracle 26ai (latest):
# cp -r confluent-new-cdc-connector-main/oracle26ai/docker/scripts .

rm -rf confluent-new-cdc-connector-main main.zip

# 3. Run Oracle container (Oracle 23ai example)
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

# 4. Wait for Oracle to start (5-10 minutes)
# Monitor progress:
docker logs -f oracle23ai
# Watch for: "DATABASE IS READY TO USE!"

# 5. Run CDC setup (once database is ready)
docker exec oracle23ai /bin/bash -c "bash /opt/oracle/scripts/setup/00_setup_cdc.sh"

# 6. Create XStream Outbound Server
docker exec -it oracle23ai sqlplus c##ggadmin/Confluent12!@FREE

SQL> -- Paste the CREATE_OUTBOUND procedure from XSTREAM_SETUP.md
SQL> -- (See the DECLARE ... BEGIN ... END block)

# 7. Test the setup
docker exec -it oracle23ai sqlplus ordermgmt/kafka@FREEPDB1

SQL> SELECT COUNT(*) FROM orders;
SQL> -- Start data generator:
SQL> BEGIN produce_orders; END;
/

# 8. Monitor XStream
docker exec -it oracle23ai sqlplus c##ggadmin/Confluent12!@FREE

SQL> SELECT CAPTURE_NAME, STATE FROM V$XSTREAM_CAPTURE;
SQL> SELECT SERVER_NAME, TOTAL_MESSAGES_SENT FROM V$XSTREAM_OUTBOUND_SERVER;
```

**Done!** Your Oracle database is now ready for CDC.

## Reference Documentation

| Document | Purpose | Time |
|----------|---------|------|
| [ORACLE_INSTALL_DOCKER.md](ORACLE_INSTALL_DOCKER.md) | Install Oracle via Docker | 10-15 min |
| [ORACLE_SETUP_STANDALONE.md](ORACLE_SETUP_STANDALONE.md) | Configure DB for CDC | 5 min |
| [XSTREAM_SETUP.md](XSTREAM_SETUP.md) | Create XStream server | 2 min |

## Support

- Oracle Container Issues: https://container-registry.oracle.com
- XStream Documentation: https://docs.oracle.com/database/
- Confluent CDC Connector: https://docs.confluent.io/

## Video Walkthrough

*(If available, add link to video demo showing the complete setup)*

---

**Ready to start?** Begin with [ORACLE_INSTALL_DOCKER.md](ORACLE_INSTALL_DOCKER.md)

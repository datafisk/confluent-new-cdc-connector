# Oracle 21c XE Standalone Setup - Quick Guide

Quick setup for Oracle 21c XE on a Linux VM with CDC configuration.

## Prerequisites

- Linux VM with Docker installed
- At least 8GB RAM, 20GB disk space
- Git repository cloned: `confluent-new-cdc-connector`

## Setup Steps (15 minutes)

### 1. Navigate to Oracle 21c Directory

```bash
cd ~/confluent-new-cdc-connector/oraclexe21c/docker
```

### 2. Start Oracle 21c Container

```bash
sudo docker run --name oracle21c \
    -p 1521:1521 -p 5500:5500 -p 8080:8080 \
    -e ORACLE_SID=XE \
    -e ORACLE_PDB=XEPDB1 \
    -e ORACLE_PWD=confluent123 \
    -e ORACLE_MEM=4000 \
    -e ORACLE_CHARACTERSET=AL32UTF8 \
    -e ENABLE_ARCHIVELOG=true \
    -v /opt/oracle/oradata \
    -v /home/confluent/confluent-new-cdc-connector/oraclexe21c/docker/scripts:/opt/oracle/scripts/setup \
    -d container-registry.oracle.com/database/express:21.3.0-xe
```

**Note:** Adjust the volume mount path if your home directory is different (replace `/home/confluent/` with your path).

### 3. Wait for Database to Start (5-10 minutes)

```bash
# Watch logs until you see "DATABASE IS READY TO USE!"
sudo docker logs -f oracle21c

# Press Ctrl+C when ready
```

### 4. Verify Scripts Are Mounted

```bash
sudo docker exec oracle21c bash -c "ls -la /opt/oracle/scripts/setup/"

# Should show:
# 00_setup_cdc.sh
# 01_setup_database.sql
# 02_create_user.sql
# 03_create_schema_datamodel.sql
# 04_load_data.sql
# 05_21c_create_user.sql
# 05_21c_privs.sql
# 06_data_generator.sql
```

### 5. Run CDC Configuration Script

```bash
# Make script executable on host
chmod +x scripts/00_setup_cdc.sh

# Run the setup (takes ~2 minutes)
sudo docker exec oracle21c bash /opt/oracle/scripts/setup/00_setup_cdc.sh
```

This script will:
- Enable archivelog mode
- Configure redo logs (2GB each)
- Enable supplemental logging
- Create users: `ordermgmt`, `c##ggadmin`, `c##cfltuser`
- Create schema with 17 tables
- Load sample data
- Create data generator procedures

### 6. Verify Installation

```bash
# Connect to Oracle
sudo docker exec -it oracle21c sqlplus sys/confluent123@XE as sysdba
```

Inside SQL*Plus:
```sql
-- Check version
SELECT banner FROM v$version;
-- Should show: Oracle Database 21c Express Edition

-- Check archivelog mode
ARCHIVE LOG LIST;
-- Should show: Database log mode: Archive Mode

-- Check users exist
SELECT username FROM dba_users WHERE username IN ('ORDERMGMT', 'C##GGADMIN');

-- Check tables
SELECT COUNT(*) FROM dba_tables WHERE owner = 'ORDERMGMT';
-- Should show: 17

EXIT;
```

## Connection Details

- **Host:** `localhost` or your VM IP
- **Port:** `1521`
- **CDB:** `XE`
- **PDB:** `XEPDB1`
- **SYS password:** `confluent123`
- **App user:** `ordermgmt` / `kafka`
- **CDC user:** `c##ggadmin` / `Confluent12!`

## Quick Connections

```bash
# SYS to CDB
sudo docker exec -it oracle21c sqlplus sys/confluent123@XE as sysdba

# SYS to PDB
sudo docker exec -it oracle21c sqlplus sys/confluent123@XEPDB1 as sysdba

# Application user
sudo docker exec -it oracle21c sqlplus ordermgmt/kafka@XEPDB1

# XStream CDC user
sudo docker exec -it oracle21c sqlplus c##ggadmin/Confluent12!@XE
```

## Optional: Enable TCPS (TLS/SSL Encryption)

For production deployments, you should encrypt CDC connections using TCPS:

📄 **[ORACLE_TCPS_SETUP.md](ORACLE_TCPS_SETUP.md)** - Complete guide to enable TLS/SSL encryption

This adds:
- Encrypted connections between connector and Oracle (port 2484)
- Certificate-based authentication
- Protection for sensitive CDC data in transit

**Recommended for:** Production environments, compliance requirements (PCI-DSS, HIPAA, SOC2)

## Next Steps

After Oracle is configured:

### 1. Create XStream Outbound Server

```bash
sudo docker exec -it oracle21c sqlplus c##ggadmin/Confluent12!@XE
```

```sql
DECLARE
  tables  DBMS_UTILITY.UNCL_ARRAY;
  schemas DBMS_UTILITY.UNCL_ARRAY;
BEGIN
  tables(1)   := 'ORDERMGMT.ORDERS';
  tables(2)   := 'ORDERMGMT.ORDER_ITEMS';
  tables(3)   := 'ORDERMGMT.CUSTOMERS';
  tables(4)   := 'ORDERMGMT.PRODUCTS';
  tables(5)   := 'ORDERMGMT.EMPLOYEES';
  tables(6)   := 'ORDERMGMT.INVENTORIES';
  tables(7)   := 'ORDERMGMT.PRODUCT_CATEGORIES';
  tables(8)   := 'ORDERMGMT.CONTACTS';
  tables(9)   := 'ORDERMGMT.NOTES';
  tables(10)  := 'ORDERMGMT.WAREHOUSES';
  tables(11)  := 'ORDERMGMT.LOCATIONS';
  tables(12)  := 'ORDERMGMT.COUNTRIES';
  tables(13)  := 'ORDERMGMT.REGIONS';
  tables(14)  := NULL;
  schemas(1)  := 'ORDERMGMT';
  
  DBMS_XSTREAM_ADM.CREATE_OUTBOUND(
    capture_name          => 'confluent_xout1',
    server_name           => 'XOUT',
    source_container_name => 'XEPDB1',
    table_names           => tables,
    schema_names          => schemas,
    connect_user          => 'C##GGADMIN',
    comment               => 'Confluent XStream CDC Connector'
  );
END;
/

EXIT;
```

**Verify it was created:**
```sql
sudo docker exec -it oracle21c sqlplus sys/confluent123@XE as sysdba
```

```sql
SELECT SERVER_NAME, CONNECT_USER, CAPTURE_NAME, SOURCE_DATABASE
FROM DBA_XSTREAM_OUTBOUND;
-- Should show: SERVER_NAME=XOUT, CONNECT_USER=C##GGADMIN, SOURCE_DATABASE=XEPDB1

EXIT;
```

### 2. Configure Confluent CDC Connector

Use this configuration in Confluent Cloud:

```json
{
  "database.hostname": "<YOUR_VM_IP>",
  "database.port": "1521",
  "database.user": "C##GGADMIN",
  "database.password": "Confluent12!",
  "database.dbname": "XE",
  "database.service.name": "XE",
  "database.pdb.name": "XEPDB1",
  "database.out.server.name": "XOUT",
  "topic.prefix": "XEPDB1",
  "table.include.list": "ORDERMGMT.ORDERS,ORDERMGMT.ORDER_ITEMS,ORDERMGMT.CUSTOMERS,ORDERMGMT.PRODUCTS"
}
```

**Important:** The `connect_user => 'C##GGADMIN'` parameter in the CREATE_OUTBOUND call is required so the connector can attach to the outbound server.

### 3. Test the CDC Flow

**Generate test data to verify CDC is working:**

```bash
sudo docker exec -it oracle21c sqlplus ordermgmt/kafka@XEPDB1
```

**Option 1: Insert a single test order**
```sql
-- Insert a test order
INSERT INTO orders (customer_id, status, salesman_id, order_date) 
VALUES (100, 'Pending', 50, SYSDATE);
COMMIT;

-- Check it was inserted
SELECT order_id, customer_id, status, order_date FROM orders ORDER BY order_id DESC FETCH FIRST 5 ROWS ONLY;
```

**Option 2: Run continuous data generator (recommended)**
```sql
-- Generates orders every 5 seconds for 5 hours
BEGIN produce_orders; END;
/

-- This will run in foreground. Press Ctrl+C to stop.
-- Or EXIT and it will stop.
```

**Option 3: Run generator in background**
```bash
# Exit sqlplus first, then run:
sudo docker exec oracle21c bash -c "nohup sqlplus ordermgmt/kafka@XEPDB1 <<EOF > /tmp/produce_orders.log 2>&1 &
BEGIN produce_orders; END;
/
EXIT;
EOF"

# Check it's running
sudo docker exec oracle21c bash -c "ps aux | grep sqlplus"

# View logs
sudo docker exec oracle21c tail -f /tmp/produce_orders.log
```

**Verify data is flowing to Confluent Cloud:**
- Go to Confluent Cloud UI → Topics
- Look for topics like `XEPDB1.ORDERMGMT.ORDERS`
- You should see messages appearing

**Monitor XStream latency:**
```bash
sudo docker exec -it oracle21c sqlplus c##ggadmin/Confluent12!@XE
```

```sql
-- Check capture latency (should be < 10 seconds)
SELECT CAPTURE_NAME, 
       ROUND((SYSDATE - CAPTURE_MESSAGE_CREATE_TIME)*86400, 2) as LATENCY_SEC
FROM V$XSTREAM_CAPTURE;

-- Check messages sent to connector
SELECT SERVER_NAME, 
       TOTAL_TRANSACTIONS_SENT,
       TOTAL_MESSAGES_SENT,
       ROUND((BYTES_SENT/1024)/1024, 2) as MB_SENT
FROM V$XSTREAM_OUTBOUND_SERVER;

EXIT;
```

## Container Management

```bash
# Stop Oracle
sudo docker stop oracle21c

# Start Oracle
sudo docker start oracle21c

# View logs
sudo docker logs oracle21c

# Remove container (WARNING: deletes data unless using named volume)
sudo docker stop oracle21c
sudo docker rm oracle21c
```

## Troubleshooting

**Container won't start:**
```bash
# Check logs
sudo docker logs oracle21c

# Check system resources
free -h
df -h
```

**Scripts not visible:**
```bash
# Verify scripts on host
ls -la ~/confluent-new-cdc-connector/oraclexe21c/docker/scripts/

# Restart container with correct path
sudo docker stop oracle21c
sudo docker rm oracle21c
# Then run step 2 again with correct absolute path
```

**Permission errors on setup script:**
```bash
# Make executable on host
chmod +x ~/confluent-new-cdc-connector/oraclexe21c/docker/scripts/00_setup_cdc.sh

# Run without -c flag
sudo docker exec oracle21c bash /opt/oracle/scripts/setup/00_setup_cdc.sh
```

**Redo log creation errors (ORA-00301 with /FREE/ path):**

If you see errors creating redo logs with `/opt/oracle/oradata/FREE/` path, the script had the wrong path. Fix manually:

```bash
sudo docker exec -it oracle21c sqlplus sys/confluent123@XE as sysdba
```

```sql
-- Add correct 2GB redo logs for XE
ALTER DATABASE ADD LOGFILE GROUP 4 '/opt/oracle/oradata/XE/redo04.log' SIZE 2G;
ALTER DATABASE ADD LOGFILE GROUP 5 '/opt/oracle/oradata/XE/redo05.log' SIZE 2G;
ALTER DATABASE ADD LOGFILE GROUP 6 '/opt/oracle/oradata/XE/redo06.log' SIZE 2G;

-- Switch to new logs
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM SWITCH LOGFILE;

-- Wait 10 seconds, then drop old 200MB logs
ALTER DATABASE DROP LOGFILE GROUP 1;
ALTER DATABASE DROP LOGFILE GROUP 2;
ALTER DATABASE DROP LOGFILE GROUP 3;

-- Verify you have 2GB logs
SELECT a.group#, b.member, a.bytes/1024/1024 as MB, a.status 
FROM v$log a, v$logfile b WHERE a.group# = b.group#;

EXIT;
```

**Connector cannot attach to outbound server:**

Error message:
```
The database 'C##GGADMIN' user does not have sufficient privileges 
to attach to the outbound server 'XOUT'.
```

**Fix:** The outbound server was created without the `connect_user` parameter. Recreate it:

```bash
sudo docker exec -it oracle21c sqlplus c##ggadmin/Confluent12!@XE
```

```sql
-- Drop existing outbound server
BEGIN
  DBMS_XSTREAM_ADM.DROP_OUTBOUND(server_name => 'XOUT');
END;
/

-- Recreate with connect_user parameter (see step 1 in Next Steps section)
-- Must include: connect_user => 'C##GGADMIN'
```

**No data appearing in Confluent Cloud topics:**

1. **Check connector status:**
   - Confluent Cloud UI → Connectors → Check status is "Running"
   - Look for errors in connector logs

2. **Verify XStream is capturing:**
   ```sql
   -- Check capture state
   SELECT CAPTURE_NAME, STATE FROM V$XSTREAM_CAPTURE;
   -- Should show: CAPTURING CHANGES or WAITING FOR REDO
   
   -- Check if connector is attached
   SELECT SERVER_NAME, STATE FROM V$XSTREAM_OUTBOUND_SERVER;
   -- Should show: SENDING TRANSACTION or similar
   ```

3. **Verify table is in XStream capture:**
   ```sql
   SELECT schema_name, object_name 
   FROM DBA_CAPTURE_PREPARED_TABLES 
   WHERE schema_name = 'ORDERMGMT';
   -- Should show all your tables
   ```

4. **Make sure you committed the transaction:**
   ```sql
   -- Always commit after INSERT/UPDATE/DELETE
   COMMIT;
   ```

## Firewall Configuration (if accessing from remote)

```bash
# Ubuntu/Debian
sudo ufw allow 1521/tcp
sudo ufw reload

# RHEL/CentOS
sudo firewall-cmd --add-port=1521/tcp --permanent
sudo firewall-cmd --reload
```

---

**Estimated time:** 15-20 minutes total (including database startup)

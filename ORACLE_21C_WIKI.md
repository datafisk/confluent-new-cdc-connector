# Oracle 21c XE Setup - Wiki Format

## Summary
Deploy Oracle 21c Express Edition with CDC configuration on Linux VM in 15 minutes.

## Prerequisites
- Linux VM (Ubuntu/RHEL/Debian)
- Docker installed
- 8GB+ RAM, 20GB+ disk
- Repository cloned: `confluent-new-cdc-connector`

---

## Installation Steps

### Step 1: Navigate to Directory
```bash
cd ~/confluent-new-cdc-connector/oraclexe21c/docker
```

### Step 2: Start Oracle Container
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
> **Note:** Replace `/home/confluent/` with your actual home directory path

### Step 3: Monitor Startup (5-10 minutes)
```bash
sudo docker logs -f oracle21c
```
Wait for: `DATABASE IS READY TO USE!` then press `Ctrl+C`

### Step 4: Verify Scripts Mounted
```bash
sudo docker exec oracle21c bash -c "ls -la /opt/oracle/scripts/setup/"
```
Expected output: 8 files (00_setup_cdc.sh through 06_data_generator.sql)

### Step 5: Configure for CDC
```bash
chmod +x scripts/00_setup_cdc.sh
sudo docker exec oracle21c bash /opt/oracle/scripts/setup/00_setup_cdc.sh
```

### Step 6: Verify Installation
```bash
sudo docker exec -it oracle21c sqlplus sys/confluent123@XE as sysdba
```
```sql
SELECT banner FROM v$version;
ARCHIVE LOG LIST;
SELECT username FROM dba_users WHERE username IN ('ORDERMGMT', 'C##GGADMIN');
SELECT COUNT(*) FROM dba_tables WHERE owner = 'ORDERMGMT';
EXIT;
```

---

## Connection Information

| Parameter | Value |
|-----------|-------|
| Host | `localhost` or VM IP |
| Port | `1521` |
| CDB | `XE` |
| PDB | `XEPDB1` |
| SYS Password | `confluent123` |
| App User | `ordermgmt` / `kafka` |
| CDC User | `c##ggadmin` / `Confluent12!` |

---

## Quick Connect Commands

**SYS to CDB:**
```bash
sudo docker exec -it oracle21c sqlplus sys/confluent123@XE as sysdba
```

**SYS to PDB:**
```bash
sudo docker exec -it oracle21c sqlplus sys/confluent123@XEPDB1 as sysdba
```

**Application User:**
```bash
sudo docker exec -it oracle21c sqlplus ordermgmt/kafka@XEPDB1
```

**XStream CDC User:**
```bash
sudo docker exec -it oracle21c sqlplus c##ggadmin/Confluent12!@XE
```

---

## Container Management

**Stop:**
```bash
sudo docker stop oracle21c
```

**Start:**
```bash
sudo docker start oracle21c
```

**View Logs:**
```bash
sudo docker logs oracle21c
```

**Remove (destroys data):**
```bash
sudo docker stop oracle21c
sudo docker rm oracle21c
```

---

## What Gets Configured

| Component | Details |
|-----------|---------|
| Archivelog Mode | Enabled |
| Redo Logs | 3x 2GB logs |
| Supplemental Logging | ALL COLUMNS enabled |
| Users Created | `ordermgmt`, `c##ggadmin`, `c##cfltuser` |
| Schema | 17 tables (orders, customers, products, etc.) |
| Sample Data | Initial dataset loaded |
| Data Generators | Stored procedures for test data |

---

## Next Steps

### Create XStream Outbound Server

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

**Verify:**
```sql
SELECT SERVER_NAME, CONNECT_USER, SOURCE_DATABASE FROM DBA_XSTREAM_OUTBOUND;
```

### Configure Confluent CDC Connector

**Connector Configuration:**
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

**Critical:** Must include `connect_user => 'C##GGADMIN'` when creating outbound server, otherwise connector cannot attach.

### Generate Test Data

**Single test insert:**
```bash
sudo docker exec -it oracle21c sqlplus ordermgmt/kafka@XEPDB1
```

```sql
INSERT INTO orders (customer_id, status, salesman_id, order_date) 
VALUES (100, 'Pending', 50, SYSDATE);
COMMIT;

SELECT order_id, customer_id, status FROM orders ORDER BY order_id DESC FETCH FIRST 5 ROWS ONLY;
EXIT;
```

**Continuous data generator:**
```bash
sudo docker exec -it oracle21c sqlplus ordermgmt/kafka@XEPDB1
```

```sql
-- Generates orders every 5 seconds for 5 hours
BEGIN produce_orders; END;
/
-- Press Ctrl+C to stop
```

**Monitor CDC flow:**
```bash
sudo docker exec -it oracle21c sqlplus c##ggadmin/Confluent12!@XE
```

```sql
-- Check latency
SELECT CAPTURE_NAME, 
       ROUND((SYSDATE - CAPTURE_MESSAGE_CREATE_TIME)*86400, 2) as LATENCY_SEC
FROM V$XSTREAM_CAPTURE;

-- Check messages sent
SELECT SERVER_NAME, TOTAL_MESSAGES_SENT 
FROM V$XSTREAM_OUTBOUND_SERVER;

EXIT;
```

**Verify in Confluent Cloud:**
- Navigate to Topics in Confluent Cloud UI
- Look for `XEPDB1.ORDERMGMT.ORDERS`
- Messages should be streaming in

---

## Common Issues

**Scripts Not Found:**
- Verify path in `-v` mount matches your home directory
- Check: `ls -la ~/confluent-new-cdc-connector/oraclexe21c/docker/scripts/`

**Permission Denied on Setup Script:**
- Run: `chmod +x scripts/00_setup_cdc.sh` on host
- Use: `bash /opt/oracle/scripts/setup/00_setup_cdc.sh` (no `-c` flag)

**Redo Log Errors (ORA-00301 with /FREE/ path):**
- Script had wrong path for Oracle 21c XE (fixed in latest version)
- Manual fix if you see this error:
  ```sql
  -- Connect as sysdba
  sqlplus sys/confluent123@XE as sysdba
  
  -- Add correct redo logs
  ALTER DATABASE ADD LOGFILE GROUP 4 '/opt/oracle/oradata/XE/redo04.log' SIZE 2G;
  ALTER DATABASE ADD LOGFILE GROUP 5 '/opt/oracle/oradata/XE/redo05.log' SIZE 2G;
  ALTER DATABASE ADD LOGFILE GROUP 6 '/opt/oracle/oradata/XE/redo06.log' SIZE 2G;
  
  -- Switch logfiles 3 times
  ALTER SYSTEM SWITCH LOGFILE;
  ALTER SYSTEM SWITCH LOGFILE;
  ALTER SYSTEM SWITCH LOGFILE;
  
  -- Drop old 200MB logs (when INACTIVE)
  ALTER DATABASE DROP LOGFILE GROUP 1;
  ALTER DATABASE DROP LOGFILE GROUP 2;
  ALTER DATABASE DROP LOGFILE GROUP 3;
  ```

**Container Won't Start:**
- Check resources: `free -h` and `df -h`
- View logs: `sudo docker logs oracle21c`

**Remote Connection Fails:**
- Ubuntu: `sudo ufw allow 1521/tcp`
- RHEL: `sudo firewall-cmd --add-port=1521/tcp --permanent`

**Connector Cannot Attach to Outbound Server:**
```
Error: The database 'C##GGADMIN' user does not have sufficient privileges to attach to outbound server 'XOUT'
```
**Fix:** Recreate outbound server with `connect_user` parameter:
```sql
-- Drop and recreate
CONNECT c##ggadmin/Confluent12!@XE
BEGIN
  DBMS_XSTREAM_ADM.DROP_OUTBOUND(server_name => 'XOUT');
END;
/
-- Then run CREATE_OUTBOUND with connect_user => 'C##GGADMIN'
```

**No Data Flowing to Topics:**

Check these in order:

1. Connector running: Check Confluent Cloud UI → Connectors → Status = "Running"
2. XStream capturing: `SELECT STATE FROM V$XSTREAM_CAPTURE;` → "CAPTURING CHANGES"
3. Connector attached: `SELECT STATE FROM V$XSTREAM_OUTBOUND_SERVER;` → "SENDING TRANSACTION"
4. Tables in capture: `SELECT * FROM DBA_CAPTURE_PREPARED_TABLES WHERE schema_name='ORDERMGMT';`
5. Transaction committed: Always run `COMMIT;` after INSERT/UPDATE/DELETE

---

## Verification Checklist

- [ ] Container running: `sudo docker ps | grep oracle21c`
- [ ] Database ready: `DATABASE IS READY TO USE!` in logs
- [ ] Scripts visible: 8 files in `/opt/oracle/scripts/setup/`
- [ ] Setup complete: All SQL scripts executed without errors
- [ ] Archivelog enabled: `ARCHIVE LOG LIST` shows "Archive Mode"
- [ ] Users created: `ORDERMGMT` and `C##GGADMIN` exist
- [ ] Tables created: 16 tables in `ORDERMGMT` schema
- [ ] Can connect: `sqlplus ordermgmt/kafka@XEPDB1` works
- [ ] XStream outbound server created: `XOUT` exists with `CONNECT_USER=C##GGADMIN`
- [ ] Connector running: Status shows "Running" in Confluent Cloud
- [ ] Topics created: `XEPDB1.ORDERMGMT.ORDERS` etc. exist in Confluent Cloud
- [ ] Data flowing: Insert test row and verify it appears in topic

---

## Time Breakdown

| Step | Duration |
|------|----------|
| Container startup | 5-10 min |
| CDC configuration | 2-3 min |
| Verification | 1 min |
| **Total** | **15-20 min** |

---

## Tags
`oracle` `cdc` `xstream` `confluent` `docker` `21c` `database`

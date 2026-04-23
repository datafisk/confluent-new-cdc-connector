# Oracle Database Setup for XStream CDC (Standalone)

This guide extracts the Oracle database configuration needed to set up an Oracle database for use with Confluent's XStream CDC Connector. Use this on any Linux VM where you have Oracle installed.

> **Note:** If you don't have Oracle installed yet, see [ORACLE_INSTALL_DOCKER.md](ORACLE_INSTALL_DOCKER.md) for complete installation instructions using Docker.

## Prerequisites

- Oracle Database installed (XE 21c, FREE 23ai/26ai, or EE 19c)
- SYS password set (examples use `confluent123` - change as needed)
- SSH or console access to the Oracle server

## Service Names by Oracle Version

Update commands based on your Oracle version:

- **Oracle 21c XE:** CDB=`XE`, PDB=`XEPDB1`
- **Oracle 23ai/26ai FREE:** CDB=`FREE`, PDB=`FREEPDB1`
- **Oracle 19c EE:** CDB=`ORCLCDB`, PDB=`ORCLPDB1`

**Important:** Replace `XE` and `XEPDB1` in the scripts below with your actual CDB/PDB names.

## Quick Setup Script

Save this as `setup_oracle_cdc.sh` and run it:

```bash
#!/bin/bash
# Oracle XStream CDC Setup Script
# Replace XE/XEPDB1 with your CDB/PDB names

# Configuration
export ORACLE_SID=XE              # Change to FREE or ORCLCDB for other versions
CDB_NAME="XE"                     # Change to FREE or ORCLCDB
PDB_NAME="XEPDB1"                 # Change to FREEPDB1 or ORCLPDB1
SYS_PASSWORD="confluent123"       # Change to your SYS password
ORDERMGMT_PASSWORD="kafka"
GGADMIN_PASSWORD="Confluent12!"

# Script directory
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

echo "=== Oracle XStream CDC Setup ==="
echo "CDB: $CDB_NAME"
echo "PDB: $PDB_NAME"
echo ""

# Step 1: Configure database for CDC
echo "[1/6] Configuring database (redo logs, archivelog, supplemental logging)..."
sqlplus -S sys/${SYS_PASSWORD}@${CDB_NAME} as sysdba @${SCRIPT_DIR}/01_setup_database.sql

# Step 2: Create application user
echo "[2/6] Creating ordermgmt user..."
sqlplus -S sys/${SYS_PASSWORD}@${PDB_NAME} as sysdba @${SCRIPT_DIR}/02_create_user.sql

# Step 3: Create schema and tables
echo "[3/6] Creating schema and data model..."
sqlplus -S ordermgmt/${ORDERMGMT_PASSWORD}@${PDB_NAME} @${SCRIPT_DIR}/03_create_schema_datamodel.sql

# Step 4: Load initial data
echo "[4/6] Loading initial data..."
sqlplus -S ordermgmt/${ORDERMGMT_PASSWORD}@${PDB_NAME} @${SCRIPT_DIR}/04_load_data.sql

# Step 5: Create data generator procedures
echo "[5/6] Creating data generator procedures..."
sqlplus -S ordermgmt/${ORDERMGMT_PASSWORD}@${PDB_NAME} @${SCRIPT_DIR}/06_data_generator.sql

# Step 6: Create CDC users (common users for container database)
echo "[6/6] Creating XStream users (c##ggadmin, c##cfltuser)..."
sqlplus -S sys/${SYS_PASSWORD}@${CDB_NAME} as sysdba @${SCRIPT_DIR}/05_21c_create_user.sql
sqlplus -S sys/${SYS_PASSWORD}@${PDB_NAME} as sysdba @${SCRIPT_DIR}/05_21c_privs.sql

echo ""
echo "=== Setup Complete ==="
echo ""
echo "Next Steps:"
echo "1. Create XStream Outbound Server (see XSTREAM_SETUP.md)"
echo "2. Configure and start Confluent XStream CDC Connector"
echo ""
echo "Credentials:"
echo "  SYS:        sys/${SYS_PASSWORD}@${CDB_NAME}"
echo "  App User:   ordermgmt/${ORDERMGMT_PASSWORD}@${PDB_NAME}"
echo "  CDC User:   c##ggadmin/${GGADMIN_PASSWORD}@${CDB_NAME}"
echo ""
```

Make it executable and run:
```bash
chmod +x setup_oracle_cdc.sh
./setup_oracle_cdc.sh
```

## Individual SQL Scripts

If you prefer to run each step manually, create these SQL files:

### 01_setup_database.sql

Configures redo logs, archivelog mode, and supplemental logging.

```sql
-- Connect as sysdba
CONNECT sys/confluent123 AS SYSDBA

-- Enable GoldenGate replication
ALTER SYSTEM SET enable_goldengate_replication=TRUE SCOPE=BOTH;

-- List current redo logs
SET LINES 200
COLUMN MEMBER FORMAT A40
COLUMN MEMBERS FORMAT 999
SELECT a.group#, b.member, a.members, a.bytes/1024/1024 as MB, a.status 
FROM v$log a, v$logfile b 
WHERE a.group# = b.group#;

-- Add new 2GB redo logs (adjust path for your Oracle version)
-- For XE: /opt/oracle/oradata/XE/
-- For FREE: /opt/oracle/oradata/FREE/
-- For ORCLCDB: /opt/oracle/oradata/ORCLCDB/
ALTER DATABASE ADD LOGFILE GROUP 4 '/opt/oracle/oradata/XE/redo04.log' SIZE 2G;
ALTER DATABASE ADD LOGFILE GROUP 5 '/opt/oracle/oradata/XE/redo05.log' SIZE 2G;
ALTER DATABASE ADD LOGFILE GROUP 6 '/opt/oracle/oradata/XE/redo06.log' SIZE 2G;

-- Switch to new redo logs
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM SWITCH LOGFILE;

-- Archive old redo logs
ALTER SYSTEM ARCHIVE LOG GROUP 1;
ALTER SYSTEM ARCHIVE LOG GROUP 2;
ALTER SYSTEM ARCHIVE LOG GROUP 3;

-- Wait and drop old redo logs
EXECUTE sys.dbms_lock.sleep(60);
ALTER DATABASE DROP LOGFILE GROUP 1;
ALTER DATABASE DROP LOGFILE GROUP 2;
ALTER DATABASE DROP LOGFILE GROUP 3;

-- Verify new redo logs
SELECT a.group#, b.member, a.members, a.bytes/1024/1024 as MB, a.status 
FROM v$log a, v$logfile b 
WHERE a.group# = b.group#;

-- Enable archivelog mode
SHUTDOWN IMMEDIATE
STARTUP MOUNT
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- Enable supplemental logging at CDB level
ALTER SESSION SET CONTAINER=cdb$root;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;

-- Enable supplemental logging at PDB level (change XEPDB1 to your PDB name)
ALTER SESSION SET CONTAINER=XEPDB1;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;

-- Make logging asynchronous for better performance
ALTER SYSTEM SET commit_logging = 'BATCH' CONTAINER=ALL;
ALTER SYSTEM SET commit_wait = 'NOWAIT' CONTAINER=ALL;

EXIT;
```

### 02_create_user.sql

Creates the application user `ordermgmt`.

```sql
-- Connect to PDB as sysdba
-- Run as: sqlplus sys/confluent123@XEPDB1 as sysdba

-- Create application user
CREATE USER ordermgmt IDENTIFIED BY kafka;

-- Grant privileges
GRANT RESOURCE TO ordermgmt;
GRANT CREATE SESSION TO ordermgmt;
GRANT EXECUTE ON DBMS_LOCK TO ordermgmt;
GRANT EXECUTE ON sys.dbms_crypto TO ORDERMGMT;

-- Give quota on USERS tablespace
ALTER USER ordermgmt QUOTA 50M ON USERS;
-- For larger datasets, use UNLIMITED:
-- ALTER USER ordermgmt QUOTA UNLIMITED ON USERS;

EXIT;
```

### 03_create_schema_datamodel.sql

Creates the order management schema (regions, countries, orders, products, etc.).

```sql
-- Connect as ordermgmt user
-- Run as: sqlplus ordermgmt/kafka@XEPDB1

-- Regions table
CREATE TABLE regions (
    region_id NUMBER(10,0) GENERATED BY DEFAULT AS IDENTITY START WITH 5 PRIMARY KEY,
    region_name VARCHAR2(50) NOT NULL
);

-- Countries table
CREATE TABLE countries (
    country_id CHAR(2) PRIMARY KEY,
    country_name VARCHAR2(40) NOT NULL,
    region_id NUMBER(10,0),
    CONSTRAINT fk_countries_regions FOREIGN KEY(region_id)
        REFERENCES regions(region_id) ON DELETE CASCADE
);

-- Locations table
CREATE TABLE locations (
    location_id NUMBER(10,0) GENERATED BY DEFAULT AS IDENTITY START WITH 24 PRIMARY KEY,
    address VARCHAR2(255) NOT NULL,
    postal_code VARCHAR2(20),
    city VARCHAR2(50),
    state VARCHAR2(50),
    country_id CHAR(2),
    CONSTRAINT fk_locations_countries FOREIGN KEY(country_id)
        REFERENCES countries(country_id) ON DELETE CASCADE
);

-- Warehouses table
CREATE TABLE warehouses (
    warehouse_id NUMBER(10,0) GENERATED BY DEFAULT AS IDENTITY START WITH 10 PRIMARY KEY,
    warehouse_name VARCHAR(255),
    location_id NUMBER(12, 0),
    CONSTRAINT fk_warehouses_locations FOREIGN KEY(location_id)
        REFERENCES locations(location_id) ON DELETE CASCADE
);

-- Employees table
CREATE TABLE employees (
    employee_id NUMBER(10,0) GENERATED BY DEFAULT AS IDENTITY START WITH 108 PRIMARY KEY,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50) NOT NULL,
    hire_date DATE NOT NULL,
    manager_id NUMBER(12, 0),
    job_title VARCHAR(255) NOT NULL,
    CONSTRAINT fk_employees_manager FOREIGN KEY(manager_id)
        REFERENCES employees(employee_id) ON DELETE CASCADE
);

-- Product categories table
CREATE TABLE product_categories (
    category_id NUMBER(10,0) GENERATED BY DEFAULT AS IDENTITY START WITH 6 PRIMARY KEY,
    category_name VARCHAR2(255) NOT NULL
);

-- Products table
CREATE TABLE products (
    product_id NUMBER(19,0) GENERATED BY DEFAULT AS IDENTITY START WITH 289 PRIMARY KEY,
    product_name VARCHAR2(255) NOT NULL,
    description VARCHAR2(2000),
    standard_cost NUMBER(9, 2),
    list_price NUMBER(9, 2),
    category_id NUMBER(10,0) NOT NULL,
    CONSTRAINT fk_products_categories FOREIGN KEY(category_id)
        REFERENCES product_categories(category_id) ON DELETE CASCADE
);

-- Customers table
CREATE TABLE customers (
    customer_id NUMBER(10,0) GENERATED BY DEFAULT AS IDENTITY START WITH 320 PRIMARY KEY,
    name VARCHAR2(255) NOT NULL,
    address VARCHAR2(255),
    website VARCHAR2(255),
    credit_limit NUMBER(8, 2)
);

-- Contacts table
CREATE TABLE contacts (
    contact_id NUMBER(10,0) GENERATED BY DEFAULT AS IDENTITY START WITH 320 PRIMARY KEY,
    first_name VARCHAR2(255) NOT NULL,
    last_name VARCHAR2(255) NOT NULL,
    email VARCHAR2(255) NOT NULL,
    phone VARCHAR2(20),
    customer_id NUMBER(10,0),
    CONSTRAINT fk_contacts_customers FOREIGN KEY(customer_id)
        REFERENCES customers(customer_id) ON DELETE CASCADE
);

-- Orders table
CREATE TABLE orders (
    order_id NUMBER(10,0) GENERATED BY DEFAULT AS IDENTITY START WITH 106 PRIMARY KEY,
    customer_id NUMBER(6, 0) NOT NULL,
    status VARCHAR(20) NOT NULL,
    salesman_id NUMBER(6, 0),
    order_date DATE NOT NULL,
    CONSTRAINT fk_orders_customers FOREIGN KEY(customer_id)
        REFERENCES customers(customer_id) ON DELETE CASCADE,
    CONSTRAINT fk_orders_employees FOREIGN KEY(salesman_id)
        REFERENCES employees(employee_id) ON DELETE SET NULL
);

-- Order items table
CREATE TABLE order_items (
    order_id NUMBER(12, 0),
    item_id NUMBER(12, 0),
    product_id NUMBER(19, 0) NOT NULL,
    quantity NUMBER(8, 2) NOT NULL,
    unit_price NUMBER(8, 2) NOT NULL,
    CONSTRAINT pk_order_items PRIMARY KEY(order_id, item_id),
    CONSTRAINT fk_order_items_products FOREIGN KEY(product_id)
        REFERENCES products(product_id) ON DELETE CASCADE,
    CONSTRAINT fk_order_items_orders FOREIGN KEY(order_id)
        REFERENCES orders(order_id) ON DELETE CASCADE
);

-- Inventories table
CREATE TABLE inventories (
    product_id NUMBER(19, 0),
    warehouse_id NUMBER(12, 0),
    quantity NUMBER(8, 0) NOT NULL,
    CONSTRAINT pk_inventories PRIMARY KEY(product_id, warehouse_id),
    CONSTRAINT fk_inventories_products FOREIGN KEY(product_id)
        REFERENCES products(product_id) ON DELETE CASCADE,
    CONSTRAINT fk_inventories_warehouses FOREIGN KEY(warehouse_id)
        REFERENCES warehouses(warehouse_id) ON DELETE CASCADE
);

-- Notes table (CLOB example)
CREATE TABLE notes (
    note_id NUMBER(12, 0),
    note CLOB,
    CONSTRAINT pk_notes PRIMARY KEY(note_id)
);

-- Connector signal table
CREATE TABLE cflt_signals (
    id VARCHAR(42) PRIMARY KEY,
    type VARCHAR(32) NOT NULL,
    data VARCHAR(2048)
);

-- Test tables
CREATE TABLE cmtest1 (
    id INTEGER GENERATED ALWAYS AS IDENTITY,
    CMTEXT VARCHAR2(50),
    CREATED_DATE DATE,
    PRIMARY KEY (id)
);

CREATE TABLE cmtest2 (
    id INTEGER GENERATED ALWAYS AS IDENTITY,
    CMTEXT VARCHAR2(50),
    CREATED_DATE DATE,
    PRIMARY KEY (id)
);

-- UUID function
CREATE OR REPLACE FUNCTION random_uuid RETURN RAW IS
    v_uuid RAW(16);
BEGIN
    v_uuid := sys.dbms_crypto.randombytes(16);
    RETURN (utl_raw.overlay(utl_raw.bit_or(utl_raw.bit_and(utl_raw.substr(v_uuid, 7, 1), '0F'), '40'), v_uuid, 7));
END random_uuid;
/

EXIT;
```

### 04_load_data.sql

This file contains ~400KB of INSERT statements. You can either:
1. Copy from `oraclexe21c/docker/scripts/04_load_data.sql` in the original repo
2. Skip this step if you just want to test CDC without initial data
3. Use the data generator (next step) to create test data

If skipping: `touch 04_load_data.sql` and add just `EXIT;`

### 05_21c_create_user.sql

Creates the XStream CDC users as common users (c##ggadmin, c##cfltuser).

```sql
-- Create XStream users in CDB
-- Run as: sqlplus sys/confluent123@XE as sysdba

ALTER SESSION SET CONTAINER = CDB$ROOT;

-- Create c##ggadmin user (XStream administrator)
CREATE USER c##ggadmin IDENTIFIED BY "Confluent12!"
    DEFAULT TABLESPACE users
    QUOTA UNLIMITED ON users
    CONTAINER=ALL;

GRANT CREATE SESSION, SET CONTAINER TO c##ggadmin CONTAINER=ALL;
GRANT DBA TO c##ggadmin CONTAINER=ALL;

-- Grant XStream privileges
BEGIN
    DBMS_XSTREAM_AUTH.GRANT_ADMIN_PRIVILEGE(
        grantee => 'C##GGADMIN',
        privilege_type => 'CAPTURE',
        grant_select_privileges => TRUE,
        container => 'ALL'
    );
END;
/

-- Create c##cfltuser (optional, for additional connector user)
CREATE USER c##cfltuser IDENTIFIED BY "Confluent12!"
    DEFAULT TABLESPACE users
    QUOTA UNLIMITED ON users
    CONTAINER=ALL;

GRANT CREATE SESSION, SET CONTAINER TO c##cfltuser CONTAINER=ALL;
GRANT SELECT_CATALOG_ROLE TO c##cfltuser CONTAINER=ALL;
GRANT FLASHBACK ANY TABLE TO c##cfltuser CONTAINER=ALL;
GRANT SELECT ANY TABLE TO c##cfltuser CONTAINER=ALL;
GRANT LOCK ANY TABLE TO c##cfltuser CONTAINER=ALL;

EXIT;
```

### 05_21c_privs.sql

Additional privileges for PDB level.

```sql
-- Run as: sqlplus sys/confluent123@XEPDB1 as sysdba

ALTER SESSION SET CONTAINER=CDB$ROOT;
GRANT CREATE SESSION, SET CONTAINER TO c##cfltuser CONTAINER=ALL;
GRANT SELECT_CATALOG_ROLE TO c##cfltuser CONTAINER=ALL;
GRANT FLASHBACK ANY TABLE TO c##cfltuser CONTAINER=ALL;
GRANT SELECT ANY TABLE TO c##cfltuser CONTAINER=ALL;
GRANT LOCK ANY TABLE TO c##cfltuser CONTAINER=ALL;

EXIT;
```

### 06_data_generator.sql

Creates procedures to continuously generate test data.

```sql
-- Run as: sqlplus ordermgmt/kafka@XEPDB1

-- Procedure to generate orders continuously (with commit every 5 seconds)
CREATE OR REPLACE PROCEDURE produce_orders
AUTHID CURRENT_USER
AS
    l_insert LONG;
BEGIN
    -- Insert for 5 hours, every 5 seconds
    FOR x IN 1..3600 LOOP
        INSERT INTO orders (customer_id, status, salesman_id, order_date) 
        VALUES (dbms_random.value(1,300), 'Pending', dbms_random.value(1,100), sysdate);
        COMMIT;
        DBMS_LOCK.sleep(seconds => 5);
    END LOOP;
END;
/

-- Procedure to generate orders without commit (for long-running transaction testing)
CREATE OR REPLACE PROCEDURE produce_orders_wo_commit
AUTHID CURRENT_USER
AS
    l_insert LONG;
BEGIN
    -- Insert 600 records in 10 minutes without commit
    FOR x IN 1..600 LOOP
        INSERT INTO orders (customer_id, status, salesman_id, order_date) 
        VALUES (dbms_random.value(1,300), 'Pending', dbms_random.value(1,100), sysdate);
        DBMS_LOCK.sleep(seconds => 1);
    END LOOP;
END;
/

EXIT;
```

## Verification

After running the setup, verify everything is configured correctly:

```bash
sqlplus sys/confluent123@XE as sysdba
```

```sql
-- Check archivelog mode
ARCHIVE LOG LIST;
-- Should show "Database log mode: Archive Mode"

-- Check supplemental logging
SELECT supplemental_log_data_min, supplemental_log_data_all 
FROM v$database;
-- Should show YES for both

-- Check redo logs
SELECT group#, bytes/1024/1024 as MB, status FROM v$log;
-- Should show 2GB logs

-- Check if users exist
SELECT username FROM dba_users WHERE username IN ('ORDERMGMT', 'C##GGADMIN', 'C##CFLTUSER');

-- Check tables
SELECT COUNT(*) FROM dba_tables WHERE owner = 'ORDERMGMT';
-- Should show 17 tables

-- Verify GoldenGate replication enabled
SHOW PARAMETER enable_goldengate_replication;
-- Should show TRUE

EXIT;
```

## Testing Data Generation

Start the data generator to create continuous orders:

```bash
sqlplus ordermgmt/kafka@XEPDB1
```

```sql
-- Start continuous order generation (runs for 5 hours)
BEGIN
    produce_orders;
END;
/
```

Or in background:
```bash
nohup sqlplus ordermgmt/kafka@XEPDB1 <<EOF > produce_orders.log 2>&1 &
BEGIN
    produce_orders;
END;
/
EXIT;
EOF
```

## Next Steps

After the database is configured, you need to:

1. **Create XStream Outbound Server** - See `XSTREAM_SETUP.md` for details
2. **Configure Confluent XStream CDC Connector** - Point it to your database
3. **Monitor XStream** - Use the monitoring SQL scripts

## Troubleshooting

**Redo log location issues:**
- Check actual path: `SELECT member FROM v$logfile;`
- Adjust paths in 01_setup_database.sql accordingly

**Memory limitations (XE edition):**
- XE is limited to 2GB RAM
- For production workloads, use Enterprise Edition

**Permission errors:**
- Ensure you're connected as SYSDBA for system changes
- Verify common user names start with `C##` for container databases

**Supplemental logging not enabled:**
```sql
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```

## File Checklist

Save all SQL files in the same directory:
- ✓ `setup_oracle_cdc.sh`
- ✓ `01_setup_database.sql`
- ✓ `02_create_user.sql`
- ✓ `03_create_schema_datamodel.sql`
- ✓ `04_load_data.sql` (optional, or copy from original repo)
- ✓ `05_21c_create_user.sql`
- ✓ `05_21c_privs.sql`
- ✓ `06_data_generator.sql`

Then run: `./setup_oracle_cdc.sh`

# Oracle XStream Outbound Server Setup

This guide shows how to create and configure the XStream Outbound Server required for the Confluent Oracle CDC Connector.

## Prerequisites

- Oracle database configured with supplemental logging (see ORACLE_SETUP_STANDALONE.md)
- User `c##ggadmin` created with XStream privileges
- Tables created in the `ORDERMGMT` schema

## Quick Setup

### Connect to Oracle

```bash
# For Oracle 21c XE
sqlplus c##ggadmin/Confluent12!@XE

# For Oracle 23ai/26ai FREE
sqlplus c##ggadmin/Confluent12!@FREE

# For Oracle 19c EE
sqlplus c##ggadmin/Confluent12!@ORCLCDB
```

### Create XStream Outbound Server

This single procedure creates the capture process, queue, and outbound server:

```sql
-- Example for XEPDB1 (Oracle 21c XE)
-- Adjust source_container_name and table list for your environment

DECLARE
    tables  DBMS_UTILITY.UNCL_ARRAY;
    schemas DBMS_UTILITY.UNCL_ARRAY;
BEGIN
    -- List all tables to capture (use schema.table format)
    tables(1)   := 'ORDERMGMT.ORDERS';
    tables(2)   := 'ORDERMGMT.ORDER_ITEMS';
    tables(3)   := 'ORDERMGMT.EMPLOYEES';
    tables(4)   := 'ORDERMGMT.PRODUCTS';
    tables(5)   := 'ORDERMGMT.CUSTOMERS';
    tables(6)   := 'ORDERMGMT.INVENTORIES';
    tables(7)   := 'ORDERMGMT.PRODUCT_CATEGORIES';
    tables(8)   := 'ORDERMGMT.CONTACTS';
    tables(9)   := 'ORDERMGMT.NOTES';
    tables(10)  := 'ORDERMGMT.WAREHOUSES';
    tables(11)  := 'ORDERMGMT.LOCATIONS';
    tables(12)  := 'ORDERMGMT.COUNTRIES';
    tables(13)  := 'ORDERMGMT.REGIONS';
    tables(14)  := NULL;  -- Array terminator
    
    -- Schema-level capture (optional, in addition to table list)
    schemas(1)  := 'ORDERMGMT';
    
    -- Create outbound server
    DBMS_XSTREAM_ADM.CREATE_OUTBOUND(
        capture_name          => 'confluent_xout1',        -- Capture process name
        server_name           => 'xout',                   -- Outbound server name
        source_container_name => 'XEPDB1',                 -- Change to FREEPDB1 or ORCLPDB1
        table_names           => tables,
        schema_names          => schemas,
        connect_user          => 'C##GGADMIN',             -- REQUIRED: User for connector to attach
        comment               => 'Confluent Xstream CDC Connector'
    );
    
    -- Set checkpoint retention (days to keep checkpoint data)
    DBMS_CAPTURE_ADM.ALTER_CAPTURE(
        capture_name => 'confluent_xout1',
        checkpoint_retention_time => 7  -- 7 days
    );
    
    -- Set capture SGA size (MB)
    -- For XE: max 256MB, for EE: can use 1024MB or more
    DBMS_XSTREAM_ADM.SET_PARAMETER(
        streams_type => 'capture',
        streams_name => 'confluent_xout1',
        parameter    => 'max_sga_size',
        value        => '256'  -- Change to '1024' for Enterprise Edition
    );
    
    -- Set outbound server SGA size (MB)
    DBMS_XSTREAM_ADM.SET_PARAMETER(
        streams_type => 'apply',
        streams_name => 'xout',
        parameter    => 'max_sga_size',
        value        => '256'  -- Change to '1024' for Enterprise Edition
    );
END;
/
```

### Version-Specific Examples

**For Oracle 23ai/26ai FREE:**
```sql
DECLARE
    tables  DBMS_UTILITY.UNCL_ARRAY;
    schemas DBMS_UTILITY.UNCL_ARRAY;
BEGIN
    tables(1)   := 'ORDERMGMT.ORDERS';
    -- ... (same table list)
    tables(14)  := NULL;
    schemas(1)  := 'ORDERMGMT';
    
    DBMS_XSTREAM_ADM.CREATE_OUTBOUND(
        capture_name          => 'confluent_xout1',
        server_name           => 'xout',
        source_container_name => 'FREEPDB1',  -- Different PDB name
        table_names           => tables,
        schema_names          => schemas,
        connect_user          => 'C##GGADMIN',
        comment               => 'Confluent Xstream CDC Connector'
    );
    
    DBMS_CAPTURE_ADM.ALTER_CAPTURE(
        capture_name => 'confluent_xout1',
        checkpoint_retention_time => 7
    );
END;
/
```

**For Oracle 19c EE:**
```sql
DECLARE
    tables  DBMS_UTILITY.UNCL_ARRAY;
    schemas DBMS_UTILITY.UNCL_ARRAY;
BEGIN
    tables(1)   := 'ORDERMGMT.ORDERS';
    -- ... (same table list)
    tables(14)  := NULL;
    schemas(1)  := 'ORDERMGMT';
    
    DBMS_XSTREAM_ADM.CREATE_OUTBOUND(
        capture_name          => 'confluent_xout1',
        server_name           => 'xout',
        source_container_name => 'ORCLPDB1',  -- Different PDB name
        table_names           => tables,
        schema_names          => schemas,
        connect_user          => 'C##GGADMIN',
        comment               => 'Confluent Xstream CDC Connector'
    );
    
    DBMS_CAPTURE_ADM.ALTER_CAPTURE(
        capture_name => 'confluent_xout1',
        checkpoint_retention_time => 7
    );
    
    -- Enterprise Edition can use more SGA
    DBMS_XSTREAM_ADM.SET_PARAMETER(
        streams_type => 'capture',
        streams_name => 'confluent_xout1',
        parameter    => 'max_sga_size',
        value        => '1024'
    );
    
    DBMS_XSTREAM_ADM.SET_PARAMETER(
        streams_type => 'apply',
        streams_name => 'xout',
        parameter    => 'max_sga_size',
        value        => '1024'
    );
END;
/
```

## Verification

After creating the outbound server, verify it was created successfully:

```sql
-- Check outbound server
SELECT SERVER_NAME, CONNECT_USER, CAPTURE_USER, CAPTURE_NAME, 
       SOURCE_DATABASE, QUEUE_OWNER, QUEUE_NAME
FROM ALL_XSTREAM_OUTBOUND;

-- Expected output:
-- SERVER_NAME: XOUT
-- CONNECT_USER: C##GGADMIN
-- CAPTURE_USER: C##GGADMIN
-- CAPTURE_NAME: CAP$_XOUT_... (auto-generated name)
-- SOURCE_DATABASE: XEPDB1 (or FREEPDB1/ORCLPDB1)

-- Check capture process status
SELECT CAPTURE_NAME, STATUS, STATE, ERROR_NUMBER
FROM DBA_CAPTURE;

-- Expected: STATUS=ENABLED, no ERROR_NUMBER

-- Check if outbound server is running
SELECT APPLY_NAME, STATUS, ERROR_NUMBER, ERROR_MESSAGE
FROM DBA_APPLY
WHERE PURPOSE = 'XStream Out';

-- Expected: STATUS=ENABLED, no errors
```

## Monitoring XStream

### Basic Status Checks

```sql
-- Check if connector is attached to outbound server
SELECT CAPTURE_NAME, STATUS 
FROM ALL_XSTREAM_OUTBOUND 
WHERE SERVER_NAME = 'XOUT';

-- Check capture latency (should be low, < 10 seconds typically)
SELECT CAPTURE_NAME,
       ((SYSDATE - CAPTURE_MESSAGE_CREATE_TIME)*86400) LATENCY_SECONDS
FROM V$XSTREAM_CAPTURE;

-- Check current transaction being processed
SELECT SERVER_NAME,
       XIDUSN ||'.'|| XIDSLT ||'.'|| XIDSQN "Transaction ID",
       COMMITSCN,
       MESSAGE_SEQUENCE
FROM V$XSTREAM_OUTBOUND_SERVER;

-- Check statistics (messages sent, bytes sent)
SELECT SERVER_NAME,
       TOTAL_TRANSACTIONS_SENT,
       TOTAL_MESSAGES_SENT,
       (BYTES_SENT/1024)/1024 as BYTES_SENT_MB
FROM V$XSTREAM_OUTBOUND_SERVER;
```

### Advanced Monitoring

For comprehensive monitoring, use the `xstream_monitor_stats.sql` script from the main repository:

```bash
sqlplus c##ggadmin/Confluent12!@XE @xstream_monitor_stats.sql
```

## Managing XStream Outbound Server

### Drop Outbound Server

If you need to recreate the outbound server:

```sql
-- Connect as c##ggadmin
EXECUTE DBMS_XSTREAM_ADM.DROP_OUTBOUND('XOUT');
```

**Warning:** This will:
- Stop the capture process
- Drop the queue
- Remove the outbound server
- You'll need to recreate it to resume CDC

### Restart After Database Restart

XStream should auto-start after database restart. Verify:

```sql
-- Check if capture is running
SELECT CAPTURE_NAME, STATE FROM V$XSTREAM_CAPTURE;
-- Expected: STATE = 'CAPTURING CHANGES' or 'WAITING FOR REDO'

-- Check if outbound server is enabled
SELECT SERVER_NAME, STATE FROM V$XSTREAM_OUTBOUND_SERVER;
-- Expected: STATE = 'SENDING TRANSACTION' or waiting for client

-- If not running, check for errors
SELECT CAPTURE_NAME, ERROR_NUMBER, ERROR_MESSAGE 
FROM DBA_CAPTURE;

SELECT APPLY_NAME, ERROR_NUMBER, ERROR_MESSAGE 
FROM DBA_APPLY 
WHERE PURPOSE = 'XStream Out';
```

## Adding Tables After Setup

To add more tables to an existing outbound server:

```sql
-- Add a single table
BEGIN
    DBMS_XSTREAM_ADM.ALTER_OUTBOUND(
        server_name => 'xout',
        table_name  => 'ORDERMGMT.NEW_TABLE',
        add_table   => TRUE
    );
END;
/

-- Or recreate the entire outbound server with new table list
-- (requires dropping first)
```

## Troubleshooting

### Connector Cannot Attach to Outbound Server

**Error:**
```
The database 'C##GGADMIN' user configured for the connector does not have 
sufficient privileges to attach to the outbound server 'XOUT'.
```

**Cause:** The outbound server was created without specifying the `connect_user` parameter.

**Fix:**
```sql
-- Connect as c##ggadmin
sqlplus c##ggadmin/Confluent12!@XE  -- or @FREE or @ORCLCDB

-- Drop the outbound server
BEGIN
  DBMS_XSTREAM_ADM.DROP_OUTBOUND(server_name => 'XOUT');
END;
/

-- Recreate with connect_user parameter (see examples above)
-- Make sure to include: connect_user => 'C##GGADMIN'
```

### Outbound Server Not Starting

```sql
-- Check for errors
SELECT ERROR_NUMBER, ERROR_MESSAGE 
FROM DBA_APPLY 
WHERE APPLY_NAME = 'XOUT';

-- Check capture process
SELECT CAPTURE_NAME, STATE, ERROR_NUMBER, ERROR_MESSAGE
FROM DBA_CAPTURE;

-- Check alert log
SELECT message_text 
FROM v$diag_alert_ext 
WHERE message_text LIKE '%XSTREAM%' 
  AND originating_timestamp > SYSDATE - 1
ORDER BY originating_timestamp DESC;
```

### High Latency

```sql
-- Check streams pool memory
SELECT TOTAL_MEMORY_ALLOCATED/1024/1024 as TOTAL_MB,
       CURRENT_SIZE/1024/1024 as CURRENT_MB
FROM V$STREAMS_POOL_STATISTICS;

-- Increase streams pool if needed (restart required)
ALTER SYSTEM SET streams_pool_size = 512M SCOPE=SPFILE;
SHUTDOWN IMMEDIATE
STARTUP
```

### Capture Process in "WAITING FOR REDO" State

This is normal when there are no changes being made. To verify it's working:

```sql
-- Make a change
INSERT INTO ordermgmt.orders (customer_id, status, salesman_id, order_date) 
VALUES (100, 'Test', 50, SYSDATE);
COMMIT;

-- Check if capture picks it up
SELECT CAPTURE_NAME, STATE, TOTAL_MESSAGES_CAPTURED 
FROM V$XSTREAM_CAPTURE;
```

### SYSAUX Tablespace Full

XStream uses SYSAUX for buffering. Monitor usage:

```sql
-- Check SYSAUX usage
SELECT tablespace_name,
       ROUND(SUM(bytes)/1048576) as USED_MB
FROM dba_segments 
WHERE tablespace_name = 'SYSAUX'
GROUP BY tablespace_name;

-- Check XStream space usage specifically
SELECT OCCUPANT_NAME, SPACE_USAGE_KBYTES/1024 as USED_MB 
FROM V$SYSAUX_OCCUPANTS 
WHERE OCCUPANT_NAME = 'STREAMS'
ORDER BY 2 DESC;

-- Add more space if needed
ALTER DATABASE DATAFILE '/path/to/sysaux01.dbf' RESIZE 2G;
```

## Performance Tuning

### For Large Transactions (millions of rows)

```sql
-- Increase checkpoint retention
DBMS_CAPTURE_ADM.ALTER_CAPTURE(
    capture_name => 'confluent_xout1',
    checkpoint_retention_time => 14  -- 14 days instead of 7
);

-- Increase SGA for capture (Enterprise Edition only)
DBMS_XSTREAM_ADM.SET_PARAMETER(
    streams_type => 'capture',
    streams_name => 'confluent_xout1',
    parameter    => 'max_sga_size',
    value        => '2048'  -- 2GB
);

-- Increase streams pool at database level
ALTER SYSTEM SET streams_pool_size = 1G SCOPE=SPFILE;
-- Requires restart
```

### Monitor Memory Usage During Large Transactions

```sql
-- Watch SGA usage
SELECT CAPTURE_NAME,
       SGA_USED/(1024*1024) AS USED_MB,
       SGA_ALLOCATED/(1024*1024) AS ALLOCATED_MB,
       TOTAL_MESSAGES_CAPTURED,
       TOTAL_MESSAGES_ENQUEUED
FROM V$XSTREAM_CAPTURE;

-- Watch LogMiner memory
SELECT SESSION_NAME,
       MAX_MEMORY_SIZE/(1024*1024) AS MAX_MB,
       USED_MEMORY_SIZE/(1024*1024) AS USED_MB,
       ROUND(USED_MEMORY_SIZE/MAX_MEMORY_SIZE * 100, 2) AS PCT_USED
FROM V$LOGMNR_SESSION;
```

## Next Steps

After XStream is configured and running:

1. **Configure Confluent Connector** - Point it to your database and outbound server
2. **Test CDC** - Make changes and verify they appear in Kafka topics
3. **Monitor** - Use XStream monitoring queries and Grafana dashboards

## Reference

- Oracle Documentation: [Oracle Streams Concepts and Administration](https://docs.oracle.com/en/database/)
- DBMS_XSTREAM_ADM package: [Oracle Package Reference](https://docs.oracle.com/en/database/oracle/oracle-database/21/arpls/DBMS_XSTREAM_ADM.html)
- Confluent XStream CDC Connector: [Confluent Documentation](https://docs.confluent.io/)

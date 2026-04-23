# Oracle Instant Client Setup for macOS (ARM64)

This guide shows how to install Oracle Instant Client on your Mac so you can run `sqlplus` and other Oracle tools directly from your terminal.

## Prerequisites

- macOS on Apple Silicon (M1/M2/M3/M4)
- Downloaded Oracle Instant Client DMG files:
  - `instantclient-basic-macos.arm64-23.26.1.0.0.dmg`
  - `instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg`

## Installation Steps

### Step 1: Install Basic Instant Client

```bash
# Mount and install the basic instant client
open instantclient-basic-macos.arm64-23.26.1.0.0.dmg

# The installer will mount and you can drag to Applications
# Or use the installer package
# Default location: /Users/$USER/Downloads/instantclient_23_26
# Or: /usr/local/instantclient_23_26
```

**Recommended installation path:** `/usr/local/instantclient_23_26`

If you need to manually install:
```bash
# Create directory
sudo mkdir -p /usr/local/instantclient_23_26

# Copy files from mounted DMG
sudo cp -R /Volumes/instantclient-basic-macos.arm64-23.26.1.0.0/* /usr/local/instantclient_23_26/

# Unmount
hdiutil unmount /Volumes/instantclient-basic-macos.arm64-23.26.1.0.0
```

### Step 2: Install SQL*Plus

```bash
# Mount and install SQL*Plus
open instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg

# Copy sqlplus files to instant client directory
sudo cp -R /Volumes/instantclient-sqlplus-macos.arm64-23.26.1.0.0/* /usr/local/instantclient_23_26/

# Unmount
hdiutil unmount /Volumes/instantclient-sqlplus-macos.arm64-23.26.1.0.0
```

### Step 3: Set Environment Variables

Add to your `~/.zshrc` (or `~/.bash_profile` if using bash):

```bash
# Oracle Instant Client
export ORACLE_HOME=/usr/local/instantclient_23_26
export DYLD_LIBRARY_PATH=$ORACLE_HOME:$DYLD_LIBRARY_PATH
export PATH=$ORACLE_HOME:$PATH
```

Apply changes:
```bash
source ~/.zshrc
```

### Step 4: Verify Installation

```bash
# Check sqlplus version
sqlplus -version

# Expected output:
# SQL*Plus: Release 23.0.0.0.0 - Production
# Version 23.26.1.0.0
```

## Usage Examples

### Connect to Local Oracle Docker Container

**Oracle 21c XE:**
```bash
# Connect as SYS to CDB
sqlplus sys/confluent123@localhost:1521/XE as sysdba

# Connect as SYS to PDB
sqlplus sys/confluent123@localhost:1521/XEPDB1 as sysdba

# Connect as application user
sqlplus ordermgmt/kafka@localhost:1521/XEPDB1

# Connect as XStream user
sqlplus c##ggadmin/Confluent12!@localhost:1521/XE
```

**Oracle 23ai/26ai FREE:**
```bash
# Connect as SYS to CDB
sqlplus sys/confluent123@localhost:1521/FREE as sysdba

# Connect as SYS to PDB
sqlplus sys/confluent123@localhost:1521/FREEPDB1 as sysdba

# Connect as application user
sqlplus ordermgmt/kafka@localhost:1521/FREEPDB1

# Connect as XStream user
sqlplus c##ggadmin/Confluent12!@localhost:1521/FREE
```

### Connect to Remote Oracle (AWS)

```bash
# Replace X.X.X.X with your Oracle server IP
sqlplus sys/confluent123@X.X.X.X:1521/XE as sysdba
sqlplus ordermgmt/kafka@X.X.X.X:1521/XEPDB1
sqlplus c##ggadmin/Confluent12!@X.X.X.X:1521/XE
```

## Shell Wrapper Scripts

Create convenient wrapper scripts for your different Oracle environments.

### Oracle 21c XE Wrappers

Save as `~/bin/sql-xe` (make executable with `chmod +x ~/bin/sql-xe`):

```bash
#!/bin/bash
# Quick connect to Oracle 21c XE CDB as sysdba

ORACLE_HOST="${ORACLE_HOST:-localhost}"
ORACLE_PORT="${ORACLE_PORT:-1521}"
ORACLE_PASSWORD="${ORACLE_PASSWORD:-confluent123}"

sqlplus sys/${ORACLE_PASSWORD}@${ORACLE_HOST}:${ORACLE_PORT}/XE as sysdba
```

Save as `~/bin/sql-xepdb1` (make executable):

```bash
#!/bin/bash
# Quick connect to Oracle 21c XE PDB as sysdba

ORACLE_HOST="${ORACLE_HOST:-localhost}"
ORACLE_PORT="${ORACLE_PORT:-1521}"
ORACLE_PASSWORD="${ORACLE_PASSWORD:-confluent123}"

sqlplus sys/${ORACLE_PASSWORD}@${ORACLE_HOST}:${ORACLE_PORT}/XEPDB1 as sysdba
```

Save as `~/bin/sql-ordermgmt` (make executable):

```bash
#!/bin/bash
# Quick connect to ordermgmt schema

ORACLE_HOST="${ORACLE_HOST:-localhost}"
ORACLE_PORT="${ORACLE_PORT:-1521}"
ORACLE_SERVICE="${ORACLE_SERVICE:-XEPDB1}"  # or FREEPDB1

sqlplus ordermgmt/kafka@${ORACLE_HOST}:${ORACLE_PORT}/${ORACLE_SERVICE}
```

Save as `~/bin/sql-ggadmin` (make executable):

```bash
#!/bin/bash
# Quick connect to XStream CDC user

ORACLE_HOST="${ORACLE_HOST:-localhost}"
ORACLE_PORT="${ORACLE_PORT:-1521}"
ORACLE_SERVICE="${ORACLE_SERVICE:-XE}"  # or FREE

sqlplus c##ggadmin/Confluent12!@${ORACLE_HOST}:${ORACLE_PORT}/${ORACLE_SERVICE}
```

### Make Scripts Executable

```bash
chmod +x ~/bin/sql-xe ~/bin/sql-xepdb1 ~/bin/sql-ordermgmt ~/bin/sql-ggadmin
```

Add `~/bin` to your PATH in `~/.zshrc`:

```bash
export PATH=$HOME/bin:$PATH
```

### Usage Examples

```bash
# Connect to local Oracle 21c XE CDB
sql-xe

# Connect to local PDB
sql-xepdb1

# Connect to application schema
sql-ordermgmt

# Connect to XStream user
sql-ggadmin

# Connect to remote Oracle (set environment variables)
ORACLE_HOST=18.185.123.45 sql-xe
ORACLE_HOST=18.185.123.45 ORACLE_SERVICE=FREEPDB1 sql-ordermgmt
```

## Advanced: TNS Names Configuration

For even easier connections, create a `tnsnames.ora` file.

Create `~/tnsnames.ora`:

```
XE_LOCAL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = XE)
    )
  )

XEPDB1_LOCAL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = XEPDB1)
    )
  )

FREE_LOCAL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = FREE)
    )
  )

FREEPDB1_LOCAL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = FREEPDB1)
    )
  )
```

Add to `~/.zshrc`:

```bash
export TNS_ADMIN=$HOME
```

Then connect using TNS aliases:

```bash
sqlplus sys/confluent123@XE_LOCAL as sysdba
sqlplus ordermgmt/kafka@XEPDB1_LOCAL
sqlplus c##ggadmin/Confluent12!@FREE_LOCAL
```

## Useful SQL Scripts

### Quick XStream Status Check

Save as `~/bin/xstream-status.sql`:

```sql
SET LINESIZE 200
SET PAGESIZE 100

PROMPT === XStream Outbound Server Status ===
SELECT SERVER_NAME, CONNECT_USER, CAPTURE_NAME, SOURCE_DATABASE, QUEUE_NAME
FROM ALL_XSTREAM_OUTBOUND;

PROMPT
PROMPT === Capture Process Status ===
SELECT CAPTURE_NAME, STATUS, STATE, ERROR_NUMBER, ERROR_MESSAGE
FROM DBA_CAPTURE;

PROMPT
PROMPT === Capture Latency (seconds) ===
SELECT CAPTURE_NAME,
       ROUND((SYSDATE - CAPTURE_MESSAGE_CREATE_TIME)*86400, 2) as LATENCY_SECONDS
FROM V$XSTREAM_CAPTURE;

PROMPT
PROMPT === Messages Sent ===
SELECT SERVER_NAME,
       TOTAL_TRANSACTIONS_SENT,
       TOTAL_MESSAGES_SENT,
       ROUND((BYTES_SENT/1024)/1024, 2) as MB_SENT
FROM V$XSTREAM_OUTBOUND_SERVER;

EXIT;
```

Run with:
```bash
sql-ggadmin @~/bin/xstream-status.sql
```

### Quick Table Row Count

Save as `~/bin/table-counts.sql`:

```sql
SET LINESIZE 200
SET PAGESIZE 100

SELECT table_name, num_rows
FROM user_tables
WHERE num_rows IS NOT NULL
ORDER BY num_rows DESC;

EXIT;
```

Run with:
```bash
sql-ordermgmt @~/bin/table-counts.sql
```

## Troubleshooting

### "dyld: Library not loaded" Error

If you see library loading errors:

```bash
# Check if DYLD_LIBRARY_PATH is set
echo $DYLD_LIBRARY_PATH

# Verify files exist
ls -l /usr/local/instantclient_23_26/

# Re-source your shell config
source ~/.zshrc
```

### "Command not found: sqlplus"

```bash
# Check if instant client is in PATH
echo $PATH | grep instantclient

# Verify sqlplus exists
ls -l /usr/local/instantclient_23_26/sqlplus

# Re-source your shell config
source ~/.zshrc
```

### Connection Errors

```bash
# Test basic connectivity to Oracle
nc -zv localhost 1521

# For remote connections:
nc -zv X.X.X.X 1521

# Check if Oracle container is running (for local Docker)
docker ps | grep oracle
```

### macOS Gatekeeper Blocking

If macOS blocks the Instant Client binaries:

```bash
# Remove quarantine attribute
sudo xattr -r -d com.apple.quarantine /usr/local/instantclient_23_26/
```

## Reference

**Connection String Formats:**

```bash
# Easy Connect format (recommended)
sqlplus user/password@host:port/service_name

# TNS alias format (requires tnsnames.ora)
sqlplus user/password@TNS_ALIAS

# Local connection (when on Oracle server)
sqlplus user/password
```

**Common Service Names:**

| Oracle Version | CDB Service | PDB Service |
|---------------|-------------|-------------|
| 21c XE        | XE          | XEPDB1      |
| 23ai FREE     | FREE        | FREEPDB1    |
| 26ai FREE     | FREE        | FREEPDB1    |
| 19c EE        | ORCLCDB     | ORCLPDB1    |

**Common Users:**

| User         | Password      | Purpose                    | Connect To |
|--------------|---------------|----------------------------|------------|
| sys          | confluent123  | Database administrator     | CDB or PDB |
| ordermgmt    | kafka         | Application schema owner   | PDB        |
| c##ggadmin   | Confluent12!  | XStream CDC administrator  | CDB        |
| c##cfltuser  | Confluent12!  | Connector user (optional)  | CDB        |

## Integration with Other Guides

- **Installing Oracle Database:** See [ORACLE_INSTALL_DOCKER.md](ORACLE_INSTALL_DOCKER.md)
- **Configuring for CDC:** See [ORACLE_SETUP_STANDALONE.md](ORACLE_SETUP_STANDALONE.md)
- **XStream Setup:** See [XSTREAM_SETUP.md](XSTREAM_SETUP.md)
- **Full Quick Start:** See [QUICKSTART_STANDALONE.md](QUICKSTART_STANDALONE.md)

## Next Steps

After setting up the client:

1. Connect to your Oracle database and verify access
2. Use the wrapper scripts to simplify daily operations
3. Create custom SQL scripts for common monitoring tasks
4. Set up tnsnames.ora for multiple environments (dev, staging, prod)

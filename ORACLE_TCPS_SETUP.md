# Oracle TCPS (TLS/SSL) Setup for CDC Connector

This guide shows how to enable encrypted TCPS connections on Oracle 21c XE and configure the Confluent CDC connector to use it.

## Overview

**TCPS (TCP with SSL/TLS)** encrypts database connections to protect sensitive CDC data in transit. This is essential for production deployments.

## Prerequisites

- Oracle 21c XE running in Docker (from ORACLE_21C_STANDALONE_QUICK.md)
- Root/sudo access to the host
- Basic understanding of SSL/TLS certificates

## Step 1: Create Oracle Wallet and Certificates

### Enter the Oracle Container

```bash
sudo docker exec -it oracle21c bash
```

### Create Wallet Directory

```bash
# Create wallet directory
mkdir -p /opt/oracle/admin/XE/wallet
cd /opt/oracle/admin/XE/wallet

# Set Oracle environment
export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE
export PATH=$ORACLE_HOME/bin:$PATH
```

### Create Oracle Wallet

```bash
# Create auto-login wallet
orapki wallet create -wallet /opt/oracle/admin/XE/wallet -auto_login

# You'll be prompted for a password - remember it!
# Example: WalletPass123!
```

### Generate Self-Signed Certificate

For testing/dev environments:

```bash
# Add self-signed certificate to wallet
orapki wallet add -wallet /opt/oracle/admin/XE/wallet \
  -dn "CN=oracle21c,OU=IT,O=YourCompany,L=YourCity,ST=YourState,C=US" \
  -keysize 2048 \
  -self_signed \
  -validity 3650 \
  -pwd WalletPass123!

# List certificates in wallet
orapki wallet display -wallet /opt/oracle/admin/XE/wallet
```

### For Production: Use CA-Signed Certificate

```bash
# 1. Create certificate request
orapki wallet add -wallet /opt/oracle/admin/XE/wallet \
  -dn "CN=your-oracle-host.company.com,OU=IT,O=Company,L=City,ST=State,C=US" \
  -keysize 2048 \
  -pwd WalletPass123!

# 2. Export certificate request
orapki wallet export -wallet /opt/oracle/admin/XE/wallet \
  -dn "CN=your-oracle-host.company.com,OU=IT,O=Company,L=City,ST=State,C=US" \
  -request /tmp/cert_request.txt \
  -pwd WalletPass123!

# 3. Send cert_request.txt to your CA (e.g., Let's Encrypt, DigiCert)
# 4. Once you receive the signed certificate, import it:
orapki wallet add -wallet /opt/oracle/admin/XE/wallet \
  -trusted_cert -cert /tmp/ca_cert.pem \
  -pwd WalletPass123!

orapki wallet add -wallet /opt/oracle/admin/XE/wallet \
  -user_cert -cert /tmp/signed_cert.pem \
  -pwd WalletPass123!
```

### Set Permissions

```bash
chmod 600 /opt/oracle/admin/XE/wallet/*
chown oracle:oinstall /opt/oracle/admin/XE/wallet/*
```

## Step 2: Configure Oracle Network Files

### Create/Update sqlnet.ora

```bash
cat > $ORACLE_HOME/network/admin/sqlnet.ora << 'EOF'
# sqlnet.ora for TCPS

# Wallet location
WALLET_LOCATION =
  (SOURCE =
    (METHOD = FILE)
    (METHOD_DATA =
      (DIRECTORY = /opt/oracle/admin/XE/wallet)
    )
  )

# SSL/TLS Configuration
SSL_CLIENT_AUTHENTICATION = FALSE
SSL_VERSION = 1.2
SSL_CIPHER_SUITES = (TLS_RSA_WITH_AES_256_CBC_SHA256, TLS_RSA_WITH_AES_128_CBC_SHA256)

# Connection timeout (optional)
SQLNET.INBOUND_CONNECT_TIMEOUT = 60
SQLNET.OUTBOUND_CONNECT_TIMEOUT = 60
EOF
```

### Update listener.ora

```bash
cat > $ORACLE_HOME/network/admin/listener.ora << 'EOF'
# listener.ora with TCPS support

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
    )
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCPS)(HOST = 0.0.0.0)(PORT = 2484))
    )
  )

# Wallet location
WALLET_LOCATION =
  (SOURCE =
    (METHOD = FILE)
    (METHOD_DATA =
      (DIRECTORY = /opt/oracle/admin/XE/wallet)
    )
  )

# SSL Configuration
SSL_CLIENT_AUTHENTICATION = FALSE

# Service definitions
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = XE)
      (ORACLE_HOME = /opt/oracle/product/21c/dbhomeXE)
      (SID_NAME = XE)
    )
  )

# Enable tracing for troubleshooting (optional)
# TRACE_LEVEL_LISTENER = SUPPORT
# TRACE_DIRECTORY_LISTENER = /opt/oracle/diag/tnslsnr
EOF
```

### Create tnsnames.ora with TCPS Entry

```bash
cat > $ORACLE_HOME/network/admin/tnsnames.ora << 'EOF'
# Regular TCP connection
XE =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = XE)
    )
  )

# TCPS encrypted connection
XE_TCPS =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCPS)(HOST = localhost)(PORT = 2484))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = XE)
    )
    (SECURITY =
      (SSL_SERVER_CERT_DN = "CN=oracle21c,OU=IT,O=YourCompany,L=YourCity,ST=YourState,C=US")
    )
  )

XEPDB1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = XEPDB1)
    )
  )

# TCPS encrypted PDB connection
XEPDB1_TCPS =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCPS)(HOST = localhost)(PORT = 2484))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = XEPDB1)
    )
    (SECURITY =
      (SSL_SERVER_CERT_DN = "CN=oracle21c,OU=IT,O=YourCompany,L=YourCity,ST=YourState,C=US")
    )
  )
EOF
```

## Step 3: Restart Oracle Listener

```bash
# Stop listener
lsnrctl stop

# Start listener
lsnrctl start

# Check listener status (should show both TCP:1521 and TCPS:2484)
lsnrctl status

# Expected output includes:
# Listening Endpoints Summary...
#   (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
#   (DESCRIPTION=(ADDRESS=(PROTOCOL=tcps)(HOST=0.0.0.0)(PORT=2484)))
```

Exit the container:
```bash
exit
```

## Step 4: Expose TCPS Port in Docker

### Stop the Container

```bash
sudo docker stop oracle21c
```

### Commit Current Container State

```bash
sudo docker commit oracle21c oracle21c-tcps:latest
```

### Remove Old Container and Recreate with TCPS Port

```bash
sudo docker rm oracle21c

sudo docker run --name oracle21c \
    -p 1521:1521 \
    -p 2484:2484 \
    -p 5500:5500 \
    -p 8080:8080 \
    -e ORACLE_SID=XE \
    -e ORACLE_PDB=XEPDB1 \
    -e ORACLE_PWD=confluent123 \
    -e ORACLE_MEM=4000 \
    -e ORACLE_CHARACTERSET=AL32UTF8 \
    -e ENABLE_ARCHIVELOG=true \
    -v /opt/oracle/oradata \
    -v /home/confluent/confluent-new-cdc-connector/oraclexe21c/docker/scripts:/opt/oracle/scripts/setup \
    -d oracle21c-tcps:latest

# Wait for database to start
sudo docker logs -f oracle21c
```

## Step 5: Test TCPS Connection

### Test from Inside Container

```bash
sudo docker exec -it oracle21c bash

# Test TCPS connection
sqlplus sys/confluent123@XE_TCPS as sysdba

# You should see:
# Connected to: Oracle Database 21c Express Edition

# Check the connection is encrypted
SQL> SELECT sys_context('USERENV', 'NETWORK_PROTOCOL') FROM dual;
-- Should show: tcps

SQL> EXIT
```

### Test from Host (if you have Oracle Instant Client)

```bash
# Copy wallet from container to host
sudo docker cp oracle21c:/opt/oracle/admin/XE/wallet /tmp/oracle_wallet

# Connect using TCPS
sqlplus sys/confluent123@"(DESCRIPTION=(ADDRESS=(PROTOCOL=TCPS)(HOST=localhost)(PORT=2484))(CONNECT_DATA=(SERVICE_NAME=XE)))" as sysdba
```

## Step 6: Configure Confluent CDC Connector for TCPS

### Update Connector Configuration

```json
{
  "connector.class": "OracleCdcSource",
  "name": "oracle-cdc-tcps-connector",
  "kafka.api.key": "${KAFKA_API_KEY}",
  "kafka.api.secret": "${KAFKA_API_SECRET}",
  
  "database.hostname": "172.200.4.14",
  "database.port": "2484",
  "database.user": "C##GGADMIN",
  "database.password": "Confluent12!",
  "database.dbname": "XE",
  "database.service.name": "XE",
  "database.pdb.name": "XEPDB1",
  "database.out.server.name": "XOUT",
  
  "connection.protocol": "TCPS",
  "connection.wallet.location": "/path/to/wallet",
  "ssl.truststore.location": "/path/to/truststore.jks",
  "ssl.truststore.password": "truststore_password",
  
  "table.include.list": "XEPDB1[.]ORDERMGMT[.].*",
  "topic.prefix": "XEPDB1",
  
  "output.data.format": "JSON_SR",
  "tasks.max": "1"
}
```

### Create Java Truststore from Oracle Wallet

If using self-managed connector, you need to convert the Oracle wallet to Java truststore:

```bash
# On the host with Oracle wallet
sudo docker exec -it oracle21c bash

# Export certificate from wallet
orapki wallet export -wallet /opt/oracle/admin/XE/wallet \
  -dn "CN=oracle21c,OU=IT,O=YourCompany,L=YourCity,ST=YourState,C=US" \
  -cert /tmp/oracle_cert.pem

# Convert to DER format
openssl x509 -in /tmp/oracle_cert.pem -out /tmp/oracle_cert.der -outform DER

# Import into Java keystore
keytool -import -alias oracle21c -keystore /tmp/truststore.jks \
  -file /tmp/oracle_cert.der -storepass changeit -noprompt

# Copy truststore to connector host
exit
sudo docker cp oracle21c:/tmp/truststore.jks /path/to/connector/truststore.jks
```

### For Fully-Managed Connector (Confluent Cloud)

Upload the certificate via Confluent Cloud UI:
1. Go to Connector configuration
2. Advanced settings → SSL/TLS
3. Upload truststore or certificate file
4. Set truststore password

## Step 7: Verify TCPS is Working

### Check Listener is Accepting TCPS

```bash
sudo docker exec oracle21c lsnrctl status | grep -A5 "Listening Endpoints"

# Should show:
# (DESCRIPTION=(ADDRESS=(PROTOCOL=tcps)(HOST=0.0.0.0)(PORT=2484)))
```

### Check Active TCPS Connections

```bash
sudo docker exec -it oracle21c sqlplus sys/confluent123@XE as sysdba
```

```sql
-- View active sessions and their protocol
SELECT username, machine, program, 
       sys_context('USERENV', 'NETWORK_PROTOCOL') as protocol
FROM v$session 
WHERE username IS NOT NULL
ORDER BY username;

-- Check for TCPS connections
SELECT COUNT(*) 
FROM v$session 
WHERE sys_context('USERENV', 'NETWORK_PROTOCOL') = 'tcps';

EXIT;
```

### Test Connector Connection

After starting the connector, check for errors:
- Confluent Cloud: Check connector status and error logs
- Self-managed: Check Connect worker logs for SSL/TLS handshake

## Troubleshooting

### Listener Not Starting with TCPS

**Check wallet permissions:**
```bash
sudo docker exec oracle21c ls -la /opt/oracle/admin/XE/wallet/
# Should show: owner=oracle, permissions=600
```

**Check listener log:**
```bash
sudo docker exec oracle21c tail -100 /opt/oracle/diag/tnslsnr/*/listener/trace/listener.log
```

### Certificate Errors

**View wallet contents:**
```bash
sudo docker exec oracle21c orapki wallet display -wallet /opt/oracle/admin/XE/wallet
```

**Check certificate DN matches:**
```bash
# In listener.ora and connector config, ensure SSL_SERVER_CERT_DN matches exactly
```

### Connection Refused on Port 2484

**Check if port is exposed:**
```bash
docker port oracle21c
# Should show: 2484/tcp -> 0.0.0.0:2484

# Test port connectivity
nc -zv localhost 2484
```

**Check firewall:**
```bash
# Ubuntu
sudo ufw allow 2484/tcp

# RHEL
sudo firewall-cmd --add-port=2484/tcp --permanent
sudo firewall-cmd --reload
```

### Connector SSL Handshake Errors

**Common issues:**

1. **Certificate CN mismatch:**
   - Ensure hostname in connector config matches certificate CN
   - Use IP address if certificate CN is IP-based

2. **Truststore not configured:**
   - Verify truststore.jks contains Oracle certificate
   - Check truststore password is correct

3. **SSL protocol version mismatch:**
   - Oracle supports TLS 1.2 by default
   - Ensure connector uses compatible TLS version

### Enable Debug Logging

**Oracle side:**
```bash
# Edit listener.ora
TRACE_LEVEL_LISTENER = SUPPORT
TRACE_DIRECTORY_LISTENER = /opt/oracle/diag/tnslsnr

# Restart listener
lsnrctl stop
lsnrctl start
```

**Connector side:**
For self-managed connector, add to connect-distributed.properties:
```properties
javax.net.debug=ssl:handshake:verbose
```

## Port Reference

| Port | Protocol | Purpose |
|------|----------|---------|
| 1521 | TCP | Standard unencrypted Oracle connections |
| 2484 | TCPS | Encrypted TLS/SSL Oracle connections |
| 5500 | HTTPS | Enterprise Manager Express |

## Security Best Practices

1. **Use CA-signed certificates in production** - Not self-signed
2. **Rotate certificates regularly** - Set validity period appropriately
3. **Disable TCP port 1521 in production** - Force TCPS only
4. **Use strong cipher suites** - TLS 1.2+ with AES-256
5. **Protect wallet files** - Strict permissions (600) and backup securely
6. **Enable certificate validation** - Set `SSL_CLIENT_AUTHENTICATION = TRUE` when ready
7. **Monitor certificate expiration** - Set up alerts before expiry

## Disable Non-Encrypted Connections (Production)

Once TCPS is working, you can disable standard TCP:

**Update listener.ora to only listen on TCPS:**
```
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCPS)(HOST = 0.0.0.0)(PORT = 2484))
    )
  )
```

**Restart listener and verify:**
```bash
lsnrctl stop
lsnrctl start
lsnrctl status
# Should only show TCPS endpoint
```

## References

- Oracle Database Security Guide: https://docs.oracle.com/en/database/oracle/oracle-database/21/dbseg/
- Oracle Net Services Administrator's Guide: https://docs.oracle.com/en/database/oracle/oracle-database/21/netag/
- Confluent Oracle CDC Connector: https://docs.confluent.io/cloud/current/connectors/cc-oracle-cdc-source.html

---

**Estimated setup time:** 30-45 minutes

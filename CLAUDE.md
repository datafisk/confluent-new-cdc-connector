# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a demonstration repository for the Confluent Oracle CDC Connector using Oracle XStream API. The repo showcases Change Data Capture (CDC) from Oracle databases (19c/21c/23ai/26ai) to Confluent Cloud, with optional sink to Oracle 23ai using DBMS_KAFKA.

**Architecture:** Multi-component system deployed via Terraform (Confluent Cloud cluster + Oracle DBs on AWS) with Docker-based self-managed connector option, monitored by Grafana/Prometheus.

**Standalone Guides (for installing Oracle on any Linux VM without AWS/Terraform):**
- `ORACLE_21C_STANDALONE_QUICK.md` - **FASTEST START** - Oracle 21c XE in 15 minutes (tested on Linux VM)
- `QUICKSTART_STANDALONE.md` - Complete 3-step guide from zero to CDC-ready
- `ORACLE_INSTALL_DOCKER.md` - Install Oracle Database via Docker (21c XE, 23ai FREE, 26ai FREE)
- `ORACLE_SETUP_STANDALONE.md` - Configure Oracle DB for CDC (redo logs, users, schema, data)
- `XSTREAM_SETUP.md` - XStream Outbound Server configuration and management
- `ORACLE_CLIENT_MACOS_SETUP.md` - Install Oracle Instant Client on macOS for local sqlplus access

## Project Structure

- **Infrastructure Terraform modules:**
  - `ccloud-cluster/` - Confluent Cloud cluster, topics, and API keys
  - `oraclexe21c/`, `oracle19c/`, `oracle23ai/`, `oracle26ai/` - Oracle DB instances on AWS EC2
  
- **Connector deployment:**
  - `cdc-connector/` - Self-managed connector via Docker Compose, includes Grafana/Prometheus monitoring
  - `cdc-connector/confluent-hub-components/` - Place for downloaded connector plugin
  - `cdc-connector/Dockerfile` - Custom Connect image with Oracle Instant Client
  
- **Documentation:**
  - `README.md` - Primary setup and deployment guide
  - `*.md` files - Specific use cases (prod_cases.md, adding_tables_automatically.md, etc.)
  - `questionnaire/` - Customer questionnaires and info gathering
  
- **SQL scripts:**
  - `xstream_monitor_stats.sql`, `simple_xstream_report.sql` - XStream monitoring
  - `DB_memory_SGA_check.sql` - Database memory analysis
  - `oraclexe21c/docker/scripts/` - Database setup, user creation, schema, data generation

## Common Development Tasks

### Initial Setup

All deployments require setting up `.accounts` file first:
```bash
# Fill in .accounts with Confluent Cloud keys, AWS keys, SSH keys, etc.
# Then source it before running terraform
source .accounts
```

### Deploying the Full Demo

**Step 1: Deploy Confluent Cloud Cluster**
```bash
cd ccloud-cluster/
source ../.accounts
terraform init
terraform plan
terraform apply
```

**Step 2: Deploy Oracle Database**
```bash
cd ../oraclexe21c/  # Or oracle19c, oracle23ai, oracle26ai
source .aws-env
terraform init
terraform plan
terraform apply
```

**Step 3a: Self-Managed Connector (Docker)**
```bash
cd ../cdc-connector
# Start Connect cluster with Grafana and Prometheus
docker-compose -f docker-compose-cdc-ccloud_new.yml up -d

# Check Connect cluster status
curl localhost:8083/ | jq
curl localhost:8083/connector-plugins/ | jq

# Create connector
curl -s -X POST -H 'Content-Type: application/json' --data @cdc_ccloud.json http://localhost:8083/connectors | jq

# Check status
curl -s -X GET -H 'Content-Type: application/json' http://localhost:8083/connectors/XSTREAMCDC0/status | jq
```

**Step 3b: Fully-Managed Connector (Confluent Cloud)**
```bash
cd ../cdc-connector
source .ccloud_env
terraform init
terraform plan
terraform apply
```

### Oracle XStream Configuration

Before starting the connector, configure the XStream outbound server in Oracle:

```bash
# SSH into Oracle instance
ssh -i ~/keys/cmawsdemoxstream.pem ec2-user@<ORACLE_IP>
sudo docker exec -it oracle21c /bin/bash  # Or oracle23ai, oracle26ai
sqlplus c##ggadmin@XE  # Password: Confluent12!

# Create outbound server (example in README.md around line 158)
# This creates capture process, queue, and outbound server
SQL> DECLARE
  tables  DBMS_UTILITY.UNCL_ARRAY;
  schemas DBMS_UTILITY.UNCL_ARRAY;
BEGIN
    tables(1)   := 'ORDERMGMT.ORDERS';
    # ... add all tables
    DBMS_XSTREAM_ADM.CREATE_OUTBOUND(
      capture_name          =>  'confluent_xout1',
      server_name           =>  'xout',
      source_container_name =>  'XEPDB1',
      table_names           =>  tables,
      schema_names          =>  schemas,
      comment               => 'Confluent Xstream CDC Connector' );
END;
/
```

### Monitoring and Operations

**Connector operations:**
```bash
# Pause connector
curl -s -X PUT -H 'Content-Type: application/json' http://localhost:8083/connectors/XSTREAMCDC0/pause | jq

# Resume connector
curl -s -X PUT -H 'Content-Type: application/json' http://localhost:8083/connectors/XSTREAMCDC0/resume | jq

# Update config
curl -s -X PUT -H 'Content-Type: application/json' --data @cdc_ccloud_update.json http://localhost:8083/connectors/XSTREAMCDC0/config | jq

# Delete connector
curl -s -X DELETE -H 'Content-Type: application/json' http://localhost:8083/connectors/XSTREAMCDC0 | jq
```

**Monitor via web UIs:**
- Grafana: http://localhost:3000 (admin/admin)
- Prometheus: http://localhost:9090
- Prometheus targets: http://localhost:9090/targets

**Oracle XStream monitoring (as c##ggadmin):**
```bash
sqlplus c##ggadmin@XE  # Password: Confluent12!

# Check if connector is attached
SQL> SELECT CAPTURE_NAME, STATUS FROM ALL_XSTREAM_OUTBOUND WHERE SERVER_NAME = 'XOUT';

# Check capture latency
SQL> SELECT CAPTURE_NAME, ((SYSDATE - CAPTURE_MESSAGE_CREATE_TIME)*86400) LATENCY_SECONDS FROM V$XSTREAM_CAPTURE;

# Full monitoring report
SQL> @xstream_monitor_stats.sql
```

### Destroying Resources

**Reverse order of creation:**
```bash
# 1. Oracle 23ai (if deployed)
cd oracle23ai/
source .aws_env
terraform destroy

# 2. CDC Connector
cd ../cdc-connector
# Self-managed:
curl -s -X DELETE -H 'Content-Type: application/json' http://localhost:8083/connectors/XSTREAMCDC0 | jq
docker-compose -f docker-compose-cdc-ccloud_new.yml down -v
# Fully-managed:
source .ccloud_env
terraform destroy

# 3. Oracle source DB
cd ../oraclexe21c
source .aws_env
terraform destroy

# 4. Confluent Cloud
cd ../ccloud-cluster
source ../.accounts
terraform destroy
```

## Key Architecture Concepts

### Oracle XStream Components

- **Outbound Server** - Streams LCRs (Logical Change Records) to the connector
- **Capture Process** - Mines redo logs for changes
- **Queue** - Buffers LCRs between capture and outbound server
- All created via single `DBMS_XSTREAM_ADM.CREATE_OUTBOUND` call

### Connector Modes

**Self-managed (Docker):**
- Runs on local desktop with Connect worker
- Uses JSON with Schema Registry format
- Includes Grafana/Prometheus monitoring
- Requires Oracle Instant Client in container (see Dockerfile)

**Fully-managed (Confluent Cloud):**
- Managed by Confluent Cloud
- Uses AVRO format
- No local infrastructure needed

### Database Service Names

Different Oracle versions use different service names:
- **21c XE:** CDB=`XE`, PDB=`XEPDB1`
- **26ai FREE:** CDB=`FREE`, PDB=`FREEPDB1`
- **19c EE:** CDB=`ORCLCDB`, PDB=`ORCLPDB1`

Update `cdc_ccloud.json.template` or `cflt_connectors.tf` accordingly.

### Performance Tuning

The connector is pre-configured for high throughput (see `cdc_ccloud.json.template`):
- `snapshot.fetch.size: 10000` - Batch size for snapshot
- `snapshot.max.threads: 4` - Parallel snapshot threads
- `max.queue.size: 65536` - Internal queue size
- `max.batch.size: 16384` - Max batch to Kafka
- `producer.override.batch.size: 204800` - Producer batching
- `producer.override.linger.ms: 50` - Producer wait time

Heap size configured in docker-compose: `-Xmx8G -Xms4G`

## Important Files

- `cdc-connector/cdc_ccloud.json.template` - Connector configuration template (gets processed by setup script)
- `cdc-connector/Dockerfile` - Custom Connect image with Oracle Instant Client for ARM64
- `cdc-connector/docker-compose-cdc-ccloud.yml.template` - Full stack: Connect, Grafana, Prometheus
- `.accounts` - Credentials file (not checked in, must be created locally)
- `xstream_monitor_stats.sql` - Comprehensive XStream monitoring report
- `prod_cases.md` - Production scenarios and troubleshooting

## Oracle Instant Client Handling

The Docker image requires Oracle Instant Client matching your hardware:
- **ARM64 (M1/M2/M3 Mac):** Uses `instantclient-basic-linux.arm64-19.26.0.0.0dbru.zip`
- **x86_64:** Different instant client URL needed
- Files `ojdbc8.jar` and `xstreams.jar` must be in `cdc-connector/confluent-hub-components/<connector>/lib/`
- `LD_LIBRARY_PATH` is set in Dockerfile to point to instant client

## Terraform Module Pattern

Each infrastructure component follows the same pattern:
1. Has its own directory with terraform files
2. Requires sourcing environment file (`.accounts`, `.aws_env`, `.ccloud_env`)
3. Standard terraform workflow: `init` → `plan` → `apply` → `destroy`
4. Outputs provide connection details and next steps

## Testing and Validation

**Verify connector is working:**
1. Check connector status via REST API
2. Monitor Grafana dashboards for metrics
3. Check Confluent Cloud UI for topics and data
4. Query Oracle: `SELECT * FROM V$XSTREAM_CAPTURE;`
5. Verify no gaps in order_id sequence in topics

**Test scenarios documented in README:**
- Large transactions (2M records)
- Long-running transactions (10 minute uncommitted)
- Pause/resume behavior
- Stop/restart recovery

## Oracle 23ai Sink (DBMS_KAFKA)

Optional: Stream data from Confluent Cloud into Oracle 23ai using built-in OSAK (Oracle SQL Access to Kafka):

```sql
-- Register Confluent Cloud cluster
DBMS_KAFKA_ADM.REGISTER_CLUSTER(
  cluster_name => 'CCLOUD1',
  BOOTSTRAP_SERVERS => 'broker:9092',
  credential_name => 'KAFKA1CRED'
);

-- Create load application
DBMS_KAFKA.CREATE_LOAD_APP('CCLOUD1', 'CCLOUDAPP', 'XEPDB1.ORDERMGMT.ORDERS');

-- Execute to sync data
DBMS_KAFKA.EXECUTE_LOAD_APP('CCLOUD1', 'CCLOUDAPP', 'MY_LOADED_DATA', v_records_inserted);
```

See README.md section 5 for full setup.

## Troubleshooting

**Common issues:**
- VPN interference with Instant Client loading (Confluent employees: disable VPN)
- Incorrect service names for different Oracle versions
- Memory limits on XE edition (max 2GB RAM)
- Missing redo logs during setup (drop old ones if present)
- ARM64 vs x86_64 Instant Client mismatch

**Debugging connector:**
```bash
# Set log level to DEBUG
curl -s -X PUT -H "Content-Type:application/json" http://localhost:8083/admin/loggers/io.confluent.connect.oracle.xstream.cdc.OracleXStreamSourceConnector -d '{"level": "DEBUG"}'

# View logs
docker-compose -f docker-compose-cdc-ccloud_new.yml logs -f connect
```

**Oracle monitoring:**
- Check SYSAUX tablespace usage: `SELECT * FROM V$SYSAUX_OCCUPANTS;`
- Monitor streams pool: `SELECT * FROM V$STREAMS_POOL_STATISTICS;`
- Check for alerts: `SELECT * FROM DBA_OUTSTANDING_ALERTS WHERE MODULE_ID LIKE '%XSTREAM%';`

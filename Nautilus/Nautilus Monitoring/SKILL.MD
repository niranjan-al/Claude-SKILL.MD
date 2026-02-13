# Nautilus Monitoring & Observability

Comprehensive monitoring, health checking, and observability guide for the Nautilus AWS Nitro Enclave auction validator system, including Kafka metrics, enclave health, and blockchain verification.

## When to Use This Skill

### Primary Triggers
- User mentions "monitor nautilus" or "check health"
- User wants to "see throughput" or "check performance"
- User asks about "kafka lag" or "consumer lag"
- User mentions "enclave status" or "is enclave running"
- User wants to "check logs" or "debug issue"
- User asks about "message processing" or "messages per second"

### Alerting Triggers
- "Setup alerts" or "create dashboard"
- "Send alerts when" or "notify me if"
- "High lag warning" or "performance degradation"

### Debugging Triggers
- "Why is throughput low" or "messages not processing"
- "Enclave crashed" or "connection issues"
- "Kafka consumer stuck" or "offset not moving"

## Core Concepts

### Nautilus Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Monitoring Layers                        │
├─────────────────────────────────────────────────────────────┤
│ 1. Enclave Health    │ /health_check, /get_attestation       │
│ 2. Kafka Metrics     │ Consumer lag, throughput, partitions  │
│ 3. System Resources  │ CPU, memory, network                  │
│ 4. Blockchain State  │ On-chain enclave status, PCRs        │
│ 5. Application Logs  │ Tracing, errors, warnings            │
└─────────────────────────────────────────────────────────────┘
```

### Key Metrics to Monitor

```yaml
Throughput:
  - Messages per second per partition (target: 6000+)
  - Total messages per second (target: 18000+ for 3 partitions)
  - Batch processing time
  - Parse/validation latency

Consumer Health:
  - Consumer lag per partition (target: < 100)
  - Offset progression
  - Partition assignment
  - Rebalances

Enclave Health:
  - Heartbeat status (every 5 seconds)
  - CPU usage (target: < 80%)
  - Memory usage (target: < 90% of 12GB)
  - Endpoint connectivity (Kafka, Sui, offset topic)

Message Quality:
  - Parse success rate (target: 100%)
  - Validation success rate
  - Signature generation rate
  - Error rate (target: < 0.1%)

System Health:
  - VSOCK connectivity
  - Socat process status
  - Network throughput
  - Disk I/O for logs
```

## Health Check Endpoints

### 1. Basic Health Check

```bash
# Basic health check
curl http://localhost:3000/health_check | jq

# Expected response:
{
  "public_key": "0x1234567890abcdef...",  # Ed25519 public key (64 hex chars)
  "endpoints": {
    "sui_testnet": "reachable",
    "kafka_broker": "reachable",
    "offset_topic": "reachable"
  }
}

# Continuous monitoring (every 5 seconds)
watch -n 5 'curl -s http://localhost:3000/health_check | jq'
```

### 2. Attestation Document Check

```bash
# Get attestation document
curl http://localhost:3000/get_attestation

# Expected: Hex-encoded attestation document
# Should be ~several KB of hex data

# Validate attestation format
ATTESTATION=$(curl -s http://localhost:3000/get_attestation)
echo "Attestation length: ${#ATTESTATION} characters"
# Should be 4000+ characters

# Parse attestation (requires AWS NSM tools)
echo "$ATTESTATION" | xxd -r -p > attestation.bin
# Verify with AWS tooling
```

### 3. Root Endpoint (Ping)

```bash
# Simple ping test
curl http://localhost:3000/

# Expected response:
"Nautilus is running!"

# Check response time
time curl -s http://localhost:3000/ > /dev/null
# Should be < 100ms
```

### 4. Automated Health Check Script

```bash
#!/bin/bash
# health_check.sh

set -euo pipefail

ENCLAVE_ENDPOINT="http://localhost:3000"
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"  # Optional

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Check basic connectivity
echo "=== Nautilus Health Check ==="
echo "Timestamp: $(date)"
echo

# 1. Ping test
echo -n "Testing root endpoint... "
if curl -sf "$ENCLAVE_ENDPOINT/" > /dev/null; then
  echo -e "${GREEN}✓${NC}"
else
  echo -e "${RED}✗ FAILED${NC}"
  exit 1
fi

# 2. Health check endpoint
echo -n "Testing health endpoint... "
HEALTH=$(curl -sf "$ENCLAVE_ENDPOINT/health_check" || echo "")
if [[ -n "$HEALTH" ]]; then
  echo -e "${GREEN}✓${NC}"
else
  echo -e "${RED}✗ FAILED${NC}"
  exit 1
fi

# 3. Parse health response
echo
echo "Health Status:"
PUBLIC_KEY=$(echo "$HEALTH" | jq -r '.public_key')
echo "  Public Key: ${PUBLIC_KEY:0:20}..."

# 4. Check endpoints
echo "  Endpoints:"
SUI_STATUS=$(echo "$HEALTH" | jq -r '.endpoints.sui_testnet // "unknown"')
KAFKA_STATUS=$(echo "$HEALTH" | jq -r '.endpoints.kafka_broker // "unknown"')
OFFSET_STATUS=$(echo "$HEALTH" | jq -r '.endpoints.offset_topic // "unknown"')

if [[ "$SUI_STATUS" == "reachable" ]]; then
  echo -e "    Sui Testnet: ${GREEN}$SUI_STATUS${NC}"
else
  echo -e "    Sui Testnet: ${RED}$SUI_STATUS${NC}"
fi

if [[ "$KAFKA_STATUS" == "reachable" ]]; then
  echo -e "    Kafka Broker: ${GREEN}$KAFKA_STATUS${NC}"
else
  echo -e "    Kafka Broker: ${RED}$KAFKA_STATUS${NC}"
fi

if [[ "$OFFSET_STATUS" == "reachable" ]]; then
  echo -e "    Offset Topic: ${GREEN}$OFFSET_STATUS${NC}"
else
  echo -e "    Offset Topic: ${RED}$OFFSET_STATUS${NC}"
fi

# 5. Overall status
echo
if [[ "$SUI_STATUS" == "reachable" && "$KAFKA_STATUS" == "reachable" && "$OFFSET_STATUS" == "reachable" ]]; then
  echo -e "${GREEN}=== All systems operational ===${NC}"
  exit 0
else
  echo -e "${RED}=== Some endpoints unreachable ===${NC}"
  
  # Send alert if webhook configured
  if [[ -n "$SLACK_WEBHOOK" ]]; then
    curl -X POST "$SLACK_WEBHOOK" \
      -H 'Content-Type: application/json' \
      -d "{\"text\":\"⚠️ Nautilus health check failed: Sui=$SUI_STATUS, Kafka=$KAFKA_STATUS, Offset=$OFFSET_STATUS\"}"
  fi
  
  exit 1
fi
```

## Kafka Monitoring

### 1. Using Kafka UI (Port 9000)

```bash
# Setup port forwarding (from local machine)
ssh -L 9000:localhost:9000 ec2-user@<parent-instance-ip>

# Open browser
# http://localhost:9000

# Key pages to monitor:
# 1. Topics → nautilus-queue-qa-v1 (input)
#    - Message rate
#    - Size
#    - Partition distribution
#
# 2. Topics → validated-auction-data-qa-v1 (output)
#    - Message rate (should match input)
#    - Lag (should be minimal)
#
# 3. Topics → nautilus-offsets-v1 (offsets)
#    - Latest offsets per partition
#    - Commit frequency
#
# 4. Consumer Groups → nautilus-consumer-group-v1
#    - Lag per partition
#    - Assigned partitions
#    - Member status
```

### 2. Consumer Lag Check

```bash
#!/bin/bash
# check_consumer_lag.sh

KAFKA_BROKER="188.42.133.76:9092"
CONSUMER_GROUP="nautilus-consumer-group-v1"
INPUT_TOPIC="nautilus-queue-qa-v1"

# Get consumer lag
kafka-consumer-groups \
  --bootstrap-server "$KAFKA_BROKER" \
  --group "$CONSUMER_GROUP" \
  --describe

# Expected output:
# GROUP                          TOPIC                PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# nautilus-consumer-group-v1    nautilus-queue-qa-v1    0       1234567        1234570         3
# nautilus-consumer-group-v1    nautilus-queue-qa-v1    1       2345678        2345680         2
# nautilus-consumer-group-v1    nautilus-queue-qa-v1    2       3456789        3456791         2

# Parse and alert on high lag
MAX_LAG=100
LAG=$(kafka-consumer-groups --bootstrap-server "$KAFKA_BROKER" \
  --group "$CONSUMER_GROUP" --describe | \
  awk 'NR>1 {print $6}' | sort -rn | head -1)

if [[ $LAG -gt $MAX_LAG ]]; then
  echo "⚠️  WARNING: Consumer lag is $LAG (threshold: $MAX_LAG)"
  exit 1
else
  echo "✓ Consumer lag is acceptable: $LAG"
fi
```

### 3. Throughput Monitoring

```bash
#!/bin/bash
# monitor_throughput.sh

KAFKA_BROKER="188.42.133.76:9092"
INPUT_TOPIC="nautilus-queue-qa-v1"
OUTPUT_TOPIC="validated-auction-data-qa-v1"
INTERVAL=10  # seconds

echo "=== Throughput Monitor ==="
echo "Measuring over ${INTERVAL}s intervals..."
echo

# Function to get message count
get_message_count() {
  local topic=$1
  kafka-run-class kafka.tools.GetOffsetShell \
    --broker-list "$KAFKA_BROKER" \
    --topic "$topic" \
    --time -1 | \
    awk -F: '{sum+=$3} END {print sum}'
}

# Initial counts
INPUT_START=$(get_message_count "$INPUT_TOPIC")
OUTPUT_START=$(get_message_count "$OUTPUT_TOPIC")

sleep "$INTERVAL"

# Final counts
INPUT_END=$(get_message_count "$INPUT_TOPIC")
OUTPUT_END=$(get_message_count "$OUTPUT_TOPIC")

# Calculate rates
INPUT_RATE=$(( (INPUT_END - INPUT_START) / INTERVAL ))
OUTPUT_RATE=$(( (OUTPUT_END - OUTPUT_START) / INTERVAL ))

echo "Input rate:  $INPUT_RATE msg/sec"
echo "Output rate: $OUTPUT_RATE msg/sec"
echo

# Check if rates match
if [[ $INPUT_RATE -eq $OUTPUT_RATE ]]; then
  echo "✓ Input and output rates match"
elif [[ $OUTPUT_RATE -lt $INPUT_RATE ]]; then
  echo "⚠️  Output rate lagging behind input"
else
  echo "⚠️  Output rate exceeds input (catching up?)"
fi

# Check against target (6000 msg/sec per partition)
MIN_RATE=5000  # Allow some variance
if [[ $OUTPUT_RATE -lt $MIN_RATE ]]; then
  echo "⚠️  WARNING: Throughput below target ($MIN_RATE msg/sec)"
else
  echo "✓ Throughput looks healthy"
fi
```

### 4. Offset Persistence Check

```bash
#!/bin/bash
# check_offset_persistence.sh

KAFKA_BROKER="188.42.133.76:9092"
OFFSET_TOPIC="nautilus-offsets-v1"

echo "=== Offset Persistence Check ==="

# Read latest offsets from offset topic
kafka-console-consumer \
  --bootstrap-server "$KAFKA_BROKER" \
  --topic "$OFFSET_TOPIC" \
  --from-beginning \
  --max-messages 10 \
  --timeout-ms 5000

# Expected format (JSON):
# {"partition":0,"offset":1234567,"timestamp":1676543210}
# {"partition":1,"offset":2345678,"timestamp":1676543210}
# {"partition":2,"offset":3456789,"timestamp":1676543210}

echo
echo "Latest offsets saved:"

# Get most recent offset per partition
for partition in 0 1 2; do
  LATEST=$(kafka-console-consumer \
    --bootstrap-server "$KAFKA_BROKER" \
    --topic "$OFFSET_TOPIC" \
    --from-beginning \
    --timeout-ms 5000 2>/dev/null | \
    grep "\"partition\":$partition" | \
    tail -1)
  
  if [[ -n "$LATEST" ]]; then
    OFFSET=$(echo "$LATEST" | jq -r '.offset')
    TIMESTAMP=$(echo "$LATEST" | jq -r '.timestamp')
    AGE=$(( $(date +%s) - TIMESTAMP ))
    echo "  Partition $partition: offset $OFFSET (${AGE}s ago)"
  else
    echo "  Partition $partition: No offset found"
  fi
done
```

## Enclave Monitoring

### 1. Enclave Status Check

```bash
# Check if enclave is running
sudo nitro-cli describe-enclaves

# Expected output:
[
  {
    "EnclaveID": "i-0abc123def456789",
    "ProcessID": 12345,
    "EnclaveCID": 88,
    "NumberOfCPUs": 4,
    "CPUIds": [1, 2, 3, 4],
    "MemoryMiB": 12288,
    "State": "RUNNING",
    "Flags": "DEBUG_MODE"
  }
]

# Check state
STATE=$(sudo nitro-cli describe-enclaves | jq -r '.[0].State')
if [[ "$STATE" == "RUNNING" ]]; then
  echo "✓ Enclave is running"
else
  echo "✗ Enclave state: $STATE"
fi
```

### 2. Enclave Resource Monitoring

```bash
#!/bin/bash
# monitor_enclave_resources.sh

# Get enclave PID
ENCLAVE_PID=$(sudo nitro-cli describe-enclaves | jq -r '.[0].ProcessID')

if [[ -z "$ENCLAVE_PID" || "$ENCLAVE_PID" == "null" ]]; then
  echo "Error: Enclave not running"
  exit 1
fi

echo "=== Enclave Resource Monitor ==="
echo "PID: $ENCLAVE_PID"
echo

# Monitor in real-time
top -b -n 1 -p "$ENCLAVE_PID" | tail -2

# Or use ps for specific metrics
CPU=$(ps -p "$ENCLAVE_PID" -o %cpu --no-headers)
MEM=$(ps -p "$ENCLAVE_PID" -o %mem --no-headers)
RSS=$(ps -p "$ENCLAVE_PID" -o rss --no-headers)

echo
echo "CPU Usage: ${CPU}%"
echo "Memory Usage: ${MEM}%"
echo "RSS: $((RSS / 1024)) MB"

# Alert thresholds
CPU_THRESHOLD=80
MEM_THRESHOLD=90

if (( $(echo "$CPU > $CPU_THRESHOLD" | bc -l) )); then
  echo "⚠️  WARNING: High CPU usage!"
fi

if (( $(echo "$MEM > $MEM_THRESHOLD" | bc -l) )); then
  echo "⚠️  WARNING: High memory usage!"
fi
```

### 3. Heartbeat Monitoring

```bash
#!/bin/bash
# monitor_heartbeat.sh

# The enclave sends heartbeat (0xB7) to VSOCK port 9000 every 5 seconds

echo "=== Heartbeat Monitor ==="
echo "Listening on VSOCK port 9000..."
echo "Expecting heartbeat every 5 seconds"
echo

# Listen for heartbeats (requires socat)
timeout 30 socat - VSOCK-LISTEN:9000,fork | xxd

# Expected output every 5 seconds:
# 00000000: b7                                       .

# Check if heartbeat received in last 10 seconds
LAST_HEARTBEAT=$(dmesg | grep -i heartbeat | tail -1)
echo "Last heartbeat: $LAST_HEARTBEAT"
```

### 4. Console Log Monitoring

```bash
# View enclave console output
sudo nitro-cli console --enclave-id <enclave-id>

# Or view from log file
sudo tail -f /var/log/nitro_enclaves/nitro_enclave.log

# Key log patterns to watch:
# - "Kafka consumer started successfully"
# - "ThroughputTracker: Processed X messages"
# - "ERROR" or "WARN" messages
# - "Connection refused" or "timeout"

# Filter for errors
sudo grep -i error /var/log/nitro_enclaves/nitro_enclave.log

# Filter for throughput stats
sudo grep -i throughput /var/log/nitro_enclaves/nitro_enclave.log | tail -20
```

## Application Log Analysis

### 1. Log Levels and Structure

```bash
# Log format (tracing crate):
# [TIMESTAMP] [LEVEL] [TARGET] MESSAGE

# Example:
# [2024-02-13T10:30:45.123Z] [INFO] [nautilus_server::kafka_consumer] Kafka consumer started successfully
# [2024-02-13T10:30:50.456Z] [WARN] [nautilus_server::common] Health check failed for sui_testnet
# [2024-02-13T10:30:55.789Z] [ERROR] [nautilus_server::kafka_producer] Failed to produce message: connection timeout
```

### 2. Log Analysis Scripts

```bash
#!/bin/bash
# analyze_logs.sh

LOG_FILE="/var/log/nitro_enclaves/nitro_enclave.log"
LOOKBACK_MINUTES=60

echo "=== Log Analysis (last $LOOKBACK_MINUTES minutes) ==="
echo

# Get timestamp from N minutes ago
SINCE=$(date -d "$LOOKBACK_MINUTES minutes ago" "+%Y-%m-%dT%H:%M")

# Count log levels
echo "Log Level Distribution:"
sudo grep "^\[" "$LOG_FILE" | \
  grep -E "\[(INFO|WARN|ERROR)\]" | \
  awk '{print $3}' | \
  sort | uniq -c | \
  awk '{printf "  %s: %d\n", $2, $1}'

echo

# Recent errors
echo "Recent Errors:"
sudo grep "^\[" "$LOG_FILE" | \
  grep "\[ERROR\]" | \
  grep "$SINCE" | \
  tail -10

echo

# Recent warnings
echo "Recent Warnings:"
sudo grep "^\[" "$LOG_FILE" | \
  grep "\[WARN\]" | \
  grep "$SINCE" | \
  tail -10

echo

# Throughput statistics
echo "Throughput Statistics:"
sudo grep "ThroughputTracker" "$LOG_FILE" | \
  grep "$SINCE" | \
  tail -5
```

### 3. Parsing Throughput Metrics

```bash
#!/bin/bash
# parse_throughput.sh

LOG_FILE="/var/log/nitro_enclaves/nitro_enclave.log"

# ThroughputTracker log format:
# [TIMESTAMP] [INFO] [nautilus_server] ThroughputTracker: Processed 1000 messages, 5 batches in 0.5s (avg parse: 0.001s, validation: 0.002s)

echo "=== Throughput Analysis ==="

# Extract latest throughput metrics
LATEST=$(sudo grep "ThroughputTracker" "$LOG_FILE" | tail -1)

if [[ -n "$LATEST" ]]; then
  MESSAGES=$(echo "$LATEST" | grep -oP "Processed \K\d+")
  BATCHES=$(echo "$LATEST" | grep -oP "\d+ batches")
  DURATION=$(echo "$LATEST" | grep -oP "in \K[\d.]+")
  PARSE_TIME=$(echo "$LATEST" | grep -oP "avg parse: \K[\d.]+")
  VALIDATION_TIME=$(echo "$LATEST" | grep -oP "validation: \K[\d.]+")
  
  RATE=$(echo "scale=2; $MESSAGES / $DURATION" | bc)
  
  echo "Latest Batch:"
  echo "  Messages: $MESSAGES"
  echo "  Batches: $BATCHES"
  echo "  Duration: ${DURATION}s"
  echo "  Rate: ${RATE} msg/sec"
  echo "  Avg Parse Time: ${PARSE_TIME}s"
  echo "  Avg Validation Time: ${VALIDATION_TIME}s"
else
  echo "No throughput data found"
fi
```

## System Resource Monitoring

### 1. VSOCK Connection Check

```bash
#!/bin/bash
# check_vsock.sh

echo "=== VSOCK Connection Check ==="

# Check socat processes
echo "Socat Processes:"
ps aux | grep socat | grep -v grep

# Expected processes:
# - Port 7777: Secret injection
# - Port 9092: Kafka broker forwarding
# - Port 3000: HTTP endpoint
# - Port 9000: Heartbeat

# Check specific ports
for port in 7777 9092 3000 9000; do
  echo -n "  Port $port: "
  if ps aux | grep "VSOCK.*:$port" | grep -v grep > /dev/null; then
    echo "✓ Active"
  else
    echo "✗ Not found"
  fi
done

# Check iptables rules for Kafka forwarding
echo
echo "IPTables Rules:"
sudo iptables -L -n -v | grep 9092
```

### 2. Network Throughput

```bash
#!/bin/bash
# monitor_network.sh

# Monitor network interface used by enclave
INTERFACE="eth0"  # Adjust as needed

echo "=== Network Throughput Monitor ==="
echo "Interface: $INTERFACE"
echo

# Get initial stats
RX_START=$(cat /sys/class/net/$INTERFACE/statistics/rx_bytes)
TX_START=$(cat /sys/class/net/$INTERFACE/statistics/tx_bytes)

sleep 10

# Get final stats
RX_END=$(cat /sys/class/net/$INTERFACE/statistics/rx_bytes)
TX_END=$(cat /sys/class/net/$INTERFACE/statistics/tx_bytes)

# Calculate rates (bytes/sec)
RX_RATE=$(( (RX_END - RX_START) / 10 ))
TX_RATE=$(( (TX_END - TX_START) / 10 ))

# Convert to human-readable
echo "RX Rate: $((RX_RATE / 1024 / 1024)) MB/s"
echo "TX Rate: $((TX_RATE / 1024 / 1024)) MB/s"

# At 6000 msg/sec, ~10KB per message = ~60 MB/s expected
MIN_EXPECTED=50  # MB/s
if [[ $((RX_RATE / 1024 / 1024)) -lt $MIN_EXPECTED ]]; then
  echo "⚠️  WARNING: Network throughput below expected"
fi
```

### 3. Disk I/O for Logs

```bash
# Monitor disk usage for logs
df -h /var/log/nitro_enclaves

# Check log file sizes
du -sh /var/log/nitro_enclaves/*

# Rotate logs if too large
LOG_SIZE=$(du -m /var/log/nitro_enclaves/nitro_enclave.log | cut -f1)
MAX_SIZE=1000  # MB

if [[ $LOG_SIZE -gt $MAX_SIZE ]]; then
  echo "⚠️  Log file exceeds ${MAX_SIZE}MB, consider rotation"
fi
```

## Blockchain State Monitoring

### 1. Check On-Chain Enclave Status

```bash
#!/bin/bash
# check_onchain_status.sh

source deployment_config.sh

echo "=== On-Chain Enclave Status ==="

# Get EnclaveConfig
CONFIG=$(sui client object "$ENCLAVE_CONFIG_ID" --json)

# Extract current enclave ID
CURRENT_ENCLAVE=$(echo "$CONFIG" | jq -r '.data.content.fields.current_enclave_id.fields.id // "null"')

if [[ "$CURRENT_ENCLAVE" == "null" ]]; then
  echo "⚠️  No enclave currently registered"
  exit 1
fi

echo "Current Enclave ID: $CURRENT_ENCLAVE"

# Get enclave details
ENCLAVE=$(sui client object "$CURRENT_ENCLAVE" --json)

PUBLIC_KEY=$(echo "$ENCLAVE" | jq -r '.data.content.fields.public_key')
CREATED_AT=$(echo "$ENCLAVE" | jq -r '.data.content.fields.created_at')

echo "Public Key: ${PUBLIC_KEY:0:40}..."
echo "Created At: $(date -d @$((CREATED_AT / 1000)) '+%Y-%m-%d %H:%M:%S')"

# Compare with running enclave
RUNNING_KEY=$(curl -s http://localhost:3000/health_check | jq -r '.public_key')

if [[ "$PUBLIC_KEY" == "$RUNNING_KEY" ]]; then
  echo "✓ On-chain key matches running enclave"
else
  echo "⚠️  Key mismatch! On-chain: ${PUBLIC_KEY:0:20}... Running: ${RUNNING_KEY:0:20}..."
fi
```

### 2. PCR Version Tracking

```bash
#!/bin/bash
# track_pcr_versions.sh

source deployment_config.sh

echo "=== PCR Version History ==="

# Get EnclaveConfig
CONFIG=$(sui client object "$ENCLAVE_CONFIG_ID" --json)

# Current version
VERSION=$(echo "$CONFIG" | jq -r '.data.content.fields.version')
echo "Current Version: $VERSION"

# Query past transactions for version updates
echo
echo "Recent PCR Updates:"
sui client events \
  --package "$ENCLAVE_PACKAGE_ID" \
  --json | \
  jq '.[] | select(.type | contains("PCRUpdated")) | {timestamp: .timestampMs, version: .parsedJson.new_version}'
```

### 3. Old Enclave Cleanup Check

```bash
#!/bin/bash
# check_old_enclaves.sh

source deployment_config.sh

echo "=== Old Enclaves Check ==="

# List all Enclave objects
ALL_ENCLAVES=$(sui client objects --filter StructType --json | \
  jq '.[] | select(.data.type | contains("Enclave"))')

COUNT=$(echo "$ALL_ENCLAVES" | jq -s 'length')

echo "Total enclaves found: $COUNT"

if [[ $COUNT -gt 1 ]]; then
  echo "⚠️  Multiple enclaves exist, consider cleanup"
  echo
  echo "Enclave IDs:"
  echo "$ALL_ENCLAVES" | jq -r '.objectId'
  echo
  echo "To clean up, run:"
  echo "  ./deploy_nautilus.sh destroy_only"
else
  echo "✓ Only one enclave exists (current)"
fi
```

## Alerting and Notifications

### 1. Slack Webhook Integration

```bash
#!/bin/bash
# send_alert.sh

SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
SEVERITY=$1  # info, warning, error
MESSAGE=$2

# Color codes
case $SEVERITY in
  info) COLOR="good";;
  warning) COLOR="warning";;
  error) COLOR="danger";;
  *) COLOR="gray";;
esac

# Send to Slack
curl -X POST "$SLACK_WEBHOOK" \
  -H 'Content-Type: application/json' \
  -d "{
    \"attachments\": [{
      \"color\": \"$COLOR\",
      \"title\": \"Nautilus Alert\",
      \"text\": \"$MESSAGE\",
      \"footer\": \"$(hostname)\",
      \"ts\": $(date +%s)
    }]
  }"
```

### 2. Email Alerts

```bash
#!/bin/bash
# send_email_alert.sh

TO="ops@example.com"
SUBJECT="$1"
BODY="$2"

echo "$BODY" | mail -s "[Nautilus Alert] $SUBJECT" "$TO"
```

### 3. Alert Conditions

```bash
#!/bin/bash
# check_alert_conditions.sh

# Define thresholds
MAX_LAG=100
MIN_THROUGHPUT=5000
MAX_ERROR_RATE=0.1  # 0.1%
MAX_CPU=80
MAX_MEM=90

# Check each condition
ALERTS=()

# 1. Consumer lag
LAG=$(kafka-consumer-groups --bootstrap-server "$KAFKA_BROKER" \
  --group "$CONSUMER_GROUP" --describe 2>/dev/null | \
  awk 'NR>1 {print $6}' | sort -rn | head -1)

if [[ $LAG -gt $MAX_LAG ]]; then
  ALERTS+=("High consumer lag: $LAG (threshold: $MAX_LAG)")
fi

# 2. Throughput
# ... (extract from logs)

# 3. Error rate
# ... (extract from logs)

# 4. CPU usage
# ... (check with ps)

# 5. Memory usage
# ... (check with ps)

# Send alerts if any
if [[ ${#ALERTS[@]} -gt 0 ]]; then
  for alert in "${ALERTS[@]}"; do
    echo "⚠️  $alert"
    ./send_alert.sh warning "$alert"
  done
fi
```

## Comprehensive Monitoring Dashboard

### 1. All-in-One Status Script

```bash
#!/bin/bash
# nautilus_status.sh

set -euo pipefail

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

clear
echo -e "${BLUE}╔════════════════════════════════════════════════════════╗${NC}"
echo -e "${BLUE}║         Nautilus Monitoring Dashboard                  ║${NC}"
echo -e "${BLUE}╚════════════════════════════════════════════════════════╝${NC}"
echo
echo "Timestamp: $(date)"
echo

# 1. Enclave Health
echo -e "${BLUE}=== Enclave Health ===${NC}"
HEALTH=$(curl -sf http://localhost:3000/health_check 2>/dev/null || echo "")
if [[ -n "$HEALTH" ]]; then
  echo -e "  Status: ${GREEN}Running${NC}"
  PUBLIC_KEY=$(echo "$HEALTH" | jq -r '.public_key')
  echo "  Public Key: ${PUBLIC_KEY:0:40}..."
  
  SUI=$(echo "$HEALTH" | jq -r '.endpoints.sui_testnet')
  KAFKA=$(echo "$HEALTH" | jq -r '.endpoints.kafka_broker')
  OFFSET=$(echo "$HEALTH" | jq -r '.endpoints.offset_topic')
  
  echo "  Endpoints:"
  [[ "$SUI" == "reachable" ]] && echo -e "    Sui: ${GREEN}✓${NC}" || echo -e "    Sui: ${RED}✗${NC}"
  [[ "$KAFKA" == "reachable" ]] && echo -e "    Kafka: ${GREEN}✓${NC}" || echo -e "    Kafka: ${RED}✗${NC}"
  [[ "$OFFSET" == "reachable" ]] && echo -e "    Offset: ${GREEN}✓${NC}" || echo -e "    Offset: ${RED}✗${NC}"
else
  echo -e "  Status: ${RED}Not responding${NC}"
fi
echo

# 2. Enclave Resources
echo -e "${BLUE}=== Enclave Resources ===${NC}"
ENCLAVE_PID=$(sudo nitro-cli describe-enclaves 2>/dev/null | jq -r '.[0].ProcessID // empty')
if [[ -n "$ENCLAVE_PID" ]]; then
  CPU=$(ps -p "$ENCLAVE_PID" -o %cpu --no-headers 2>/dev/null || echo "0")
  MEM=$(ps -p "$ENCLAVE_PID" -o %mem --no-headers 2>/dev/null || echo "0")
  RSS=$(ps -p "$ENCLAVE_PID" -o rss --no-headers 2>/dev/null || echo "0")
  
  echo "  CPU: ${CPU}%"
  echo "  Memory: ${MEM}% ($((RSS / 1024)) MB)"
else
  echo -e "  ${RED}Enclave not running${NC}"
fi
echo

# 3. Kafka Consumer Status
echo -e "${BLUE}=== Kafka Consumer ===${NC}"
CONSUMER_INFO=$(kafka-consumer-groups --bootstrap-server "$KAFKA_BROKER" \
  --group "$CONSUMER_GROUP" --describe 2>/dev/null | awk 'NR>1')

if [[ -n "$CONSUMER_INFO" ]]; then
  echo "  Partitions:"
  echo "$CONSUMER_INFO" | while read -r line; do
    PARTITION=$(echo "$line" | awk '{print $3}')
    OFFSET=$(echo "$line" | awk '{print $4}')
    LAG=$(echo "$line" | awk '{print $6}')
    
    if [[ $LAG -gt 100 ]]; then
      echo -e "    Partition $PARTITION: offset $OFFSET, lag ${RED}$LAG${NC}"
    else
      echo -e "    Partition $PARTITION: offset $OFFSET, lag ${GREEN}$LAG${NC}"
    fi
  done
else
  echo -e "  ${RED}Consumer group not found${NC}"
fi
echo

# 4. Recent Throughput
echo -e "${BLUE}=== Recent Throughput ===${NC}"
THROUGHPUT=$(sudo grep "ThroughputTracker" /var/log/nitro_enclaves/nitro_enclave.log 2>/dev/null | tail -1)
if [[ -n "$THROUGHPUT" ]]; then
  echo "  $THROUGHPUT" | grep -oP "Processed.*"
else
  echo "  No recent throughput data"
fi
echo

# 5. Recent Errors
echo -e "${BLUE}=== Recent Errors ===${NC}"
ERRORS=$(sudo grep "\[ERROR\]" /var/log/nitro_enclaves/nitro_enclave.log 2>/dev/null | tail -3)
if [[ -n "$ERRORS" ]]; then
  echo "$ERRORS" | while read -r line; do
    echo -e "  ${RED}$line${NC}"
  done
else
  echo -e "  ${GREEN}No recent errors${NC}"
fi
echo

echo -e "${BLUE}╚════════════════════════════════════════════════════════╝${NC}"
```

### 2. Continuous Monitoring Loop

```bash
#!/bin/bash
# monitor_continuous.sh

INTERVAL=30  # seconds

while true; do
  ./nautilus_status.sh
  echo
  echo "Refreshing in ${INTERVAL}s... (Ctrl+C to stop)"
  sleep "$INTERVAL"
done
```

## Troubleshooting Runbook

### Issue: Low Throughput

```bash
# Symptoms:
# - Messages per second below 5000
# - Consumer lag increasing
# - Throughput metrics dropping

# Diagnosis:
# 1. Check enclave CPU/memory
ps aux | grep enclave

# 2. Check Kafka broker connectivity
curl http://localhost:3000/health_check | jq '.endpoints.kafka_broker'

# 3. Check network throughput
iftop -i eth0

# 4. Check for errors in logs
sudo grep ERROR /var/log/nitro_enclaves/nitro_enclave.log | tail -20

# Solutions:
# - If CPU high: Increase enclave CPU allocation
# - If network issues: Check socat processes
# - If Kafka issues: Restart Kafka forwarding
# - If memory high: Consider memory optimization
```

### Issue: Consumer Lag Increasing

```bash
# Symptoms:
# - Lag per partition > 1000
# - Growing over time
# - Messages not being consumed

# Diagnosis:
# 1. Check if enclave is processing
sudo grep ThroughputTracker /var/log/nitro_enclaves/nitro_enclave.log | tail -5

# 2. Check offset commits
kafka-console-consumer --bootstrap-server "$KAFKA_BROKER" \
  --topic nautilus-offsets-v1 --from-beginning | tail -10

# 3. Check for errors
sudo grep ERROR /var/log/nitro_enclaves/nitro_enclave.log

# Solutions:
# - If not processing: Restart enclave
# - If errors: Fix underlying issue
# - If Kafka retention: Consumer from-earliest
# - If partition rebalance: Wait for stabilization
```

### Issue: Health Check Failing

```bash
# Symptoms:
# - curl http://localhost:3000/health_check times out
# - Endpoints showing "unreachable"
# - 500 errors

# Diagnosis:
# 1. Check if enclave is running
sudo nitro-cli describe-enclaves

# 2. Check VSOCK forwarding
ps aux | grep socat | grep 3000

# 3. Check enclave logs
sudo nitro-cli console --enclave-id <id>

# Solutions:
# - If enclave crashed: Restart enclave
# - If VSOCK down: Run configure_enclave.sh
# - If NSM issues: Check /dev/nsm permissions
# - If initialization: Wait 60 seconds
```

## Best Practices

### 1. Regular Health Checks
```bash
# Run health check every 5 minutes via cron
*/5 * * * * /home/ec2-user/nautilus/health_check.sh
```

### 2. Log Rotation
```bash
# Rotate logs daily to prevent disk filling
# /etc/logrotate.d/nitro-enclaves
/var/log/nitro_enclaves/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
}
```

### 3. Alert on Critical Issues
```bash
# Set up alerts for:
# - Enclave crashes
# - Consumer lag > 1000
# - Throughput < 3000 msg/sec
# - Error rate > 1%
# - Endpoints unreachable
```

### 4. Baseline Performance
```bash
# Document baseline metrics:
# - Normal throughput: 6000-7000 msg/sec per partition
# - Normal lag: < 50
# - Normal CPU: 40-60%
# - Normal memory: 4-6 GB
```

### 5. Monitor Trends
```bash
# Track metrics over time:
# - Throughput trends
# - Lag trends
# - Error rate trends
# - Resource usage trends
```

## Summary

This skill provides comprehensive monitoring for Nautilus:
- Health check endpoints and automation
- Kafka metrics and consumer monitoring
- Enclave resource tracking
- Application log analysis
- System resource monitoring
- Blockchain state verification
- Alerting and notifications
- Troubleshooting runbooks

Use this as your central reference for all monitoring and observability operations.
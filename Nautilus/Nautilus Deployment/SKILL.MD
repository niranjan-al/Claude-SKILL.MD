# Nautilus Deployment Automation

Comprehensive deployment automation for the Nautilus AWS Nitro Enclave auction validator system, including EIF builds, Sui blockchain registration, and multi-environment deployment orchestration.

## When to Use This Skill

### Primary Triggers
- User mentions "deploy nautilus" or "deploy enclave"
- User wants to "update production" or "push to QA/staging/prod"
- User asks to "build and deploy" or "create new EIF"
- User mentions "register enclave" or "update PCRs"
- User wants to "roll back" or "destroy old enclave"
- User asks about "deployment process" or "how to deploy"

### Environment-Specific Triggers
- "Deploy to dev/qa/staging/prod"
- "Update the dev/qa/staging/prod environment"
- "Switch to a new enclave in production"

### Build-Related Triggers
- "Build EIF" or "create enclave image"
- "Update PCRs" or "register new PCRs"
- "Generate attestation document"

## Core Concepts

### Nautilus Architecture
The Nautilus system consists of:
1. **Parent EC2 Instance**: Hosts the enclave, manages VSOCK communication, forwards Kafka traffic
2. **Nitro Enclave**: Isolated TEE running the validator, generates attestations
3. **Sui Blockchain**: Stores PCR values and enclave registration, verifies signatures
4. **Kafka**: Message queue for auction data (input) and validated results (output)

### Deployment Modes
Based on `deploy_nautilus.sh`:
- **update**: Build EIF, update PCRs on-chain, register new enclave, destroy old
- **no_update**: Register enclave without PCR update (reuses existing PCRs)
- **destroy_only**: Clean up old enclaves without deploying new ones

### Key Files
```
Nautilus/
├── Containerfile              # EIF build with StageX
├── deploy_nautilus.sh         # Main deployment orchestrator
├── register_enclave_enhanced.sh  # Enclave registration logic
├── configure_enclave.sh       # Enclave startup and configuration
├── deployment_config.sh       # Package IDs and environment config
├── secrets.json              # Runtime config (Kafka, topics, mode)
├── out/
│   ├── nitro.eif            # Compiled enclave image
│   └── nitro.pcrs           # PCR measurements (JSON)
└── move/enclave/
    ├── sources/enclave.move  # Sui smart contract
    └── Published.toml        # Deployed package IDs
```

### Environment Configuration
```bash
# Dev: Local development, may use testnet
# QA: 188.42.133.76:9092 (Kafka), testnet Sui
# Staging: Pre-production testing
# Prod: Production with mainnet Sui
```

## Pre-Deployment Checklist

Before running any deployment, verify:

1. **Git Branch Strategy** (New Structure)
   ```bash
   # Ensure you're on the correct branch
   - dev branch → dev environment
   - qa branch → qa environment  
   - stg branch → staging environment
   - prod branch → production environment
   ```

2. **Environment Variables**
   ```bash
   export SUI_NETWORK="testnet"  # or "mainnet"
   export KAFKA_BROKER="188.42.133.76:9092"
   export MODE="update"  # or "no_update", "destroy_only"
   ```

3. **Configuration Files**
   - `deployment_config.sh`: Package IDs, cap objects, RPC URLs
   - `secrets.json`: Kafka config, topics, consumer group

4. **Prerequisites**
   - Sui CLI installed and configured
   - Docker installed for EIF builds
   - SSH access to parent EC2 instance
   - Admin capability (Cap object) for Sui contract updates

## Deployment Workflows

### 1. Full Deployment (Build + Deploy)

**Use When**: Deploying code changes, new features, or bug fixes

```bash
# Step 1: Build EIF locally
docker build --target export -f Containerfile -o out/ .

# Step 2: Verify PCRs were generated
cat out/nitro.pcrs
# Expected output: JSON with PCR0, PCR1, PCR2

# Step 3: Update deployment_config.sh with package IDs
source deployment_config.sh
echo "ENCLAVE_PACKAGE_ID: $ENCLAVE_PACKAGE_ID"
echo "ENCLAVE_CONFIG_ID: $ENCLAVE_CONFIG_ID"

# Step 4: Run deployment (update mode)
./deploy_nautilus.sh update

# Step 5: Verify deployment
# Check health endpoint (from parent instance)
curl http://localhost:3000/health_check

# Step 6: Monitor logs
# On parent instance:
tail -f /var/log/nitro_enclaves/nitro_enclave.log
```

**What This Does**:
1. Builds EIF from Containerfile (StageX multi-stage build)
2. Extracts PCR0, PCR1, PCR2 from out/nitro.pcrs
3. Calls `update_pcrs` on Sui with new PCR values
4. Starts enclave on parent instance
5. Fetches attestation document from GET /get_attestation
6. Registers enclave on-chain via `register_enclave`
7. Destroys old enclave (if exists)

### 2. Quick Re-registration (No PCR Update)

**Use When**: Restarting enclave without code changes, or recovering from crash

```bash
# Step 1: Ensure existing EIF is available
ls -lh out/nitro.eif

# Step 2: Run deployment (no_update mode)
./deploy_nautilus.sh no_update

# Step 3: Verify registration
./register_enclave_enhanced.sh verify
```

**What This Does**:
1. Skips EIF build (uses existing out/nitro.eif)
2. Skips PCR update on-chain
3. Starts enclave with existing configuration
4. Registers new enclave instance with same PCRs
5. Does NOT destroy old enclave

### 3. Cleanup Old Enclaves

**Use When**: Removing stale enclaves to save resources or clean up after testing

```bash
# Step 1: List current enclaves
sui client objects --filter StructType --json | jq '.[] | select(.data.type | contains("Enclave"))'

# Step 2: Run destroy mode
./deploy_nautilus.sh destroy_only

# Step 3: Verify cleanup
sui client objects --filter StructType --json | jq '.[] | select(.data.type | contains("Enclave"))'
```

**What This Does**:
1. Queries on-chain for old enclaves
2. Calls `destroy_old_enclave` for each stale enclave
3. Does NOT build or register new enclave

### 4. Manual Step-by-Step Deployment

**Use When**: Debugging deployment issues or learning the process

```bash
# Step 1: Build EIF manually
docker build --target export -f Containerfile -o out/ .

# Step 2: Copy EIF to parent instance
scp out/nitro.eif ec2-user@<parent-ip>:/home/ec2-user/nautilus/

# Step 3: SSH to parent instance
ssh ec2-user@<parent-ip>

# Step 4: Stop existing enclave (if running)
sudo nitro-cli terminate-enclave --all

# Step 5: Start new enclave
cd /home/ec2-user/nautilus
sudo nitro-cli run-enclave \
  --cpu-count 4 \
  --memory 12288 \
  --eif-path out/nitro.eif \
  --enclave-cid 88 \
  --debug-mode

# Step 6: Configure VSOCK forwarding
./configure_enclave.sh

# Step 7: Inject secrets
socat - VSOCK-CONNECT:88:7777 < secrets.json

# Step 8: Get attestation
curl http://localhost:3000/get_attestation > attestation.hex

# Step 9: Register on-chain
./register_enclave_enhanced.sh register attestation.hex

# Step 10: Health check
curl http://localhost:3000/health_check | jq
```

## Environment-Specific Deployment

### Dev Environment
```bash
# Configuration
export SUI_NETWORK="testnet"
export KAFKA_BROKER="localhost:9092"  # Local Kafka for dev
export MODE="no_update"  # Faster iteration

# secrets.json
{
  "kafka_brokers": ["localhost:9092"],
  "input_topic": "nautilus-queue-dev-v1",
  "output_topic": "validated-auction-data-dev-v1",
  "consumer_group": "nautilus-consumer-group-dev-v1",
  "run_mode": "Debug"
}

# Deploy
./deploy_nautilus.sh no_update
```

### QA Environment
```bash
# Configuration
export SUI_NETWORK="testnet"
export KAFKA_BROKER="188.42.133.76:9092"
export MODE="update"

# secrets.json
{
  "kafka_brokers": ["188.42.133.76:9092"],
  "input_topic": "nautilus-queue-qa-v1",
  "output_topic": "validated-auction-data-qa-v1",
  "consumer_group": "nautilus-consumer-group-v1",
  "run_mode": "Debug",
  "offset_topic": "nautilus-offsets-v1"
}

# Deploy
./deploy_nautilus.sh update
```

### Staging Environment
```bash
# Configuration
export SUI_NETWORK="testnet"  # or dedicated staging network
export KAFKA_BROKER="<staging-broker>:9092"
export MODE="update"

# secrets.json - Similar to QA but with staging topics
{
  "kafka_brokers": ["<staging-broker>:9092"],
  "input_topic": "nautilus-queue-stg-v1",
  "output_topic": "validated-auction-data-stg-v1",
  "consumer_group": "nautilus-consumer-group-stg-v1",
  "run_mode": "Release"
}

# Deploy
./deploy_nautilus.sh update
```

### Production Environment
```bash
# Configuration
export SUI_NETWORK="mainnet"
export KAFKA_BROKER="<prod-broker>:9092"
export MODE="update"

# secrets.json - Production config
{
  "kafka_brokers": ["<prod-broker>:9092"],
  "input_topic": "nautilus-queue-prod-v1",
  "output_topic": "validated-auction-data-prod-v1",
  "consumer_group": "nautilus-consumer-group-prod-v1",
  "run_mode": "Release",
  "max_batch_size": 1000,
  "max_wait_ms": 5000
}

# Deploy (with extra caution)
# 1. Deploy to canary instance first
# 2. Monitor for 30 minutes
# 3. If stable, deploy to full fleet
./deploy_nautilus.sh update

# Post-deployment verification
curl http://localhost:3000/health_check
# Check Kafka UI for message flow
# Monitor CloudWatch for errors
```

## Rollback Procedures

### Quick Rollback (Re-register Old EIF)
```bash
# Step 1: Find old EIF (should be versioned/backed up)
ls -lh out/nitro.eif.backup-*

# Step 2: Restore old EIF
cp out/nitro.eif.backup-<timestamp> out/nitro.eif
cp out/nitro.pcrs.backup-<timestamp> out/nitro.pcrs

# Step 3: Re-deploy with no_update mode
./deploy_nautilus.sh no_update

# Step 4: Verify
curl http://localhost:3000/health_check
```

### Full Rollback (Rebuild Old Version)
```bash
# Step 1: Checkout previous Git commit/branch
git checkout <previous-commit-or-tag>

# Step 2: Rebuild EIF
docker build --target export -f Containerfile -o out/ .

# Step 3: Deploy with update mode (updates PCRs to old version)
./deploy_nautilus.sh update

# Step 4: Verify
curl http://localhost:3000/health_check
```

### Emergency Rollback (Switch to Standby Enclave)
```bash
# If you maintain a standby enclave with old version:

# Step 1: Get standby enclave ID
sui client object <standby-enclave-id>

# Step 2: Update current_enclave_id in EnclaveConfig
sui client call \
  --package $ENCLAVE_PACKAGE_ID \
  --module enclave \
  --function switch_to_standby \
  --args $ENCLAVE_CONFIG_ID $CAP_ID $STANDBY_ENCLAVE_ID

# Step 3: Verify
curl http://localhost:3000/health_check
```

## Versioning Strategy

### EIF Versioning
```bash
# Before building new EIF, backup current one
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
cp out/nitro.eif out/nitro.eif.backup-$TIMESTAMP
cp out/nitro.pcrs out/nitro.pcrs.backup-$TIMESTAMP

# Tag Git commit with version
git tag -a v1.2.3 -m "Release 1.2.3: Fixed Kafka offset bug"
git push origin v1.2.3
```

### Sui Contract Versioning
```bash
# From move/enclave/Published.toml
# Keep ORIGINAL_PACKAGE_ID fixed (immutable)
# Update ENCLAVE_PACKAGE_ID on upgrades

# Example upgrade flow:
# 1. Make contract changes
# 2. Build and publish upgrade
sui client publish --skip-fetch-latest-git-deps move/enclave

# 3. Update Published.toml with new ENCLAVE_PACKAGE_ID
# 4. Update deployment_config.sh
# 5. Re-deploy enclaves (they'll use new package ID)
```

## Common Issues and Solutions

### Issue 1: PCR Mismatch
**Symptom**: Registration fails with "PCR values don't match"
**Cause**: On-chain PCRs don't match attestation document
**Solution**:
```bash
# Step 1: Verify PCRs in attestation
curl http://localhost:3000/get_attestation | xxd | grep -A 3 "PCR"

# Step 2: Check on-chain PCRs
sui client object $ENCLAVE_CONFIG_ID --json | jq '.data.content.fields.pcr0'

# Step 3: Update PCRs if needed
./deploy_nautilus.sh update
```

### Issue 2: Enclave Won't Start
**Symptom**: `nitro-cli run-enclave` fails or times out
**Cause**: Resource constraints, bad EIF, or kernel issue
**Solution**:
```bash
# Step 1: Check enclave logs
sudo cat /var/log/nitro_enclaves/nitro_enclave.log

# Step 2: Verify EIF integrity
file out/nitro.eif
# Should show: "data"

# Step 3: Check resources
free -h  # Ensure 12GB+ RAM available
lscpu | grep CPU  # Ensure 4+ CPUs available

# Step 4: Try with reduced resources
sudo nitro-cli run-enclave --cpu-count 2 --memory 8192 --eif-path out/nitro.eif

# Step 5: Check kernel module
lsmod | grep nitro
# Should show: nitro_enclaves
```

### Issue 3: Kafka Connection Fails
**Symptom**: Health check shows Kafka endpoint as "unreachable"
**Cause**: VSOCK forwarding not configured or Kafka broker down
**Solution**:
```bash
# Step 1: Check socat processes
ps aux | grep socat
# Should show: socat TCP-LISTEN:9092,fork,reuseaddr VSOCK-CONNECT:88:9092

# Step 2: Restart VSOCK forwarding
sudo pkill socat
./configure_enclave.sh

# Step 3: Test Kafka connectivity from parent
nc -zv 188.42.133.76 9092

# Step 4: Check iptables rules
sudo iptables -L -n -v | grep 9092
```

### Issue 4: Attestation Document Fetch Fails
**Symptom**: `curl http://localhost:3000/get_attestation` returns 500 or times out
**Cause**: NSM driver issue or enclave not fully initialized
**Solution**:
```bash
# Step 1: Check if enclave is running
sudo nitro-cli describe-enclaves

# Step 2: Check enclave console logs
sudo nitro-cli console --enclave-id <id>

# Step 3: Wait for initialization (can take 30-60 seconds)
for i in {1..60}; do
  curl -s http://localhost:3000/health_check && break
  echo "Waiting... ($i/60)"
  sleep 1
done

# Step 4: If still failing, check NSM device
ls -l /dev/nsm
# Should show: crw-rw-rw- 1 root root
```

### Issue 5: Old Enclave Won't Destroy
**Symptom**: `destroy_only` mode fails with "object not found"
**Cause**: Enclave already destroyed or object ID mismatch
**Solution**:
```bash
# Step 1: List all enclaves manually
sui client objects --filter StructType --json | jq '.[] | select(.data.type | contains("Enclave"))'

# Step 2: Manually destroy specific enclave
sui client call \
  --package $ENCLAVE_PACKAGE_ID \
  --module enclave \
  --function destroy_old_enclave \
  --args $ENCLAVE_CONFIG_ID $CAP_ID <old-enclave-id>

# Step 3: Verify destruction
sui client object <old-enclave-id>
# Should show: "Object not found"
```

## Monitoring Post-Deployment

### Health Check Validation
```bash
# Basic health check
curl http://localhost:3000/health_check | jq

# Expected response:
{
  "public_key": "0x1234...",
  "endpoints": {
    "sui_testnet": "reachable",
    "kafka_broker": "reachable",
    "offset_topic": "reachable"
  }
}

# Continuous monitoring
watch -n 5 'curl -s http://localhost:3000/health_check | jq'
```

### Kafka Message Flow
```bash
# On parent instance with port forwarding
ssh -L 9000:localhost:9000 ec2-user@<parent-ip>

# Open Kafka UI in browser
# http://localhost:9000
# Check:
# - Consumer lag for nautilus-consumer-group-v1
# - Messages in validated-auction-data-qa-v1
# - Throughput (should be 6000+ msg/sec per partition)
```

### Enclave Resource Usage
```bash
# Check enclave metrics
sudo nitro-cli describe-enclaves

# Expected output:
{
  "EnclaveID": "i-abc123...",
  "CPUCount": 4,
  "MemoryMiB": 12288,
  "State": "RUNNING"
}

# Monitor CPU/memory from parent
top -p <enclave-pid>
```

### Log Analysis
```bash
# Enclave logs
sudo tail -f /var/log/nitro_enclaves/nitro_enclave.log

# Look for:
# - "Kafka consumer started successfully"
# - "ThroughputTracker: Processed X messages"
# - Warning/Error messages

# Parent instance logs
journalctl -u nitro-enclaves-allocator -f
```

## Best Practices

### 1. Always Version Your EIFs
```bash
# Before any build, backup current EIF
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
GIT_SHA=$(git rev-parse --short HEAD)
cp out/nitro.eif out/nitro.eif.$GIT_SHA.$TIMESTAMP
cp out/nitro.pcrs out/nitro.pcrs.$GIT_SHA.$TIMESTAMP
```

### 2. Test in Lower Environments First
```bash
# Deployment order: dev → qa → staging → prod
# Never skip staging for production deployments
```

### 3. Use Canary Deployments
```bash
# For production, deploy to 1 instance first
# Monitor for 30-60 minutes
# If stable, deploy to remaining fleet
```

### 4. Maintain Deployment Logs
```bash
# Log all deployments
echo "[$(date)] Deployed commit $GIT_SHA to $ENVIRONMENT" >> deployments.log
```

### 5. Keep Old Enclaves for Quick Rollback
```bash
# Don't destroy old enclave immediately
# Wait 1-2 hours to ensure new enclave is stable
# Then run: ./deploy_nautilus.sh destroy_only
```

### 6. Verify Kafka Offset Persistence
```bash
# After deployment, check that offsets are being saved
sui client events --package $ENCLAVE_PACKAGE_ID | grep OffsetCommitted

# Or check offset topic directly
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic nautilus-offsets-v1 \
  --from-beginning
```

## Automated Deployment Script Template

```bash
#!/bin/bash
# automated_nautilus_deploy.sh

set -euo pipefail

ENVIRONMENT=$1  # dev, qa, stg, prod
MODE=${2:-update}  # update, no_update, destroy_only

echo "=== Nautilus Deployment ==="
echo "Environment: $ENVIRONMENT"
echo "Mode: $MODE"
echo "Timestamp: $(date)"

# Step 1: Validate environment
if [[ ! "$ENVIRONMENT" =~ ^(dev|qa|stg|prod)$ ]]; then
  echo "Error: Invalid environment. Use: dev, qa, stg, or prod"
  exit 1
fi

# Step 2: Load environment config
source "config/${ENVIRONMENT}.env"

# Step 3: Backup current EIF
if [[ -f out/nitro.eif ]]; then
  BACKUP_NAME="out/nitro.eif.$(git rev-parse --short HEAD).$(date +%Y%m%d_%H%M%S)"
  cp out/nitro.eif "$BACKUP_NAME"
  echo "Backed up EIF to: $BACKUP_NAME"
fi

# Step 4: Run deployment
./deploy_nautilus.sh "$MODE"

# Step 5: Wait for enclave to initialize
echo "Waiting for enclave to initialize..."
for i in {1..60}; do
  if curl -s http://localhost:3000/health_check > /dev/null; then
    echo "Enclave initialized successfully!"
    break
  fi
  if [[ $i -eq 60 ]]; then
    echo "Error: Enclave failed to initialize after 60 seconds"
    exit 1
  fi
  sleep 1
done

# Step 6: Validate health check
HEALTH=$(curl -s http://localhost:3000/health_check)
echo "Health check response:"
echo "$HEALTH" | jq

# Step 7: Verify all endpoints are reachable
UNREACHABLE=$(echo "$HEALTH" | jq '.endpoints | to_entries | map(select(.value != "reachable")) | length')
if [[ "$UNREACHABLE" -gt 0 ]]; then
  echo "Warning: Some endpoints are unreachable!"
  echo "$HEALTH" | jq '.endpoints | to_entries | map(select(.value != "reachable"))'
fi

# Step 8: Log deployment
echo "[$(date)] Deployed $ENVIRONMENT with mode $MODE (commit: $(git rev-parse --short HEAD))" >> deployments.log

echo "=== Deployment Complete ==="
```

## Integration with CI/CD

### GitHub Actions Example
```yaml
name: Deploy Nautilus

on:
  push:
    branches:
      - qa
      - stg
      - prod

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build EIF
        run: docker build --target export -f Containerfile -o out/ .
      
      - name: Copy EIF to parent instance
        run: |
          scp -i ${{ secrets.SSH_KEY }} \
            out/nitro.eif \
            ec2-user@${{ secrets.PARENT_INSTANCE_IP }}:/home/ec2-user/nautilus/
      
      - name: Deploy
        run: |
          ssh -i ${{ secrets.SSH_KEY }} \
            ec2-user@${{ secrets.PARENT_INSTANCE_IP }} \
            'cd /home/ec2-user/nautilus && ./deploy_nautilus.sh update'
      
      - name: Health Check
        run: |
          ssh -i ${{ secrets.SSH_KEY }} \
            ec2-user@${{ secrets.PARENT_INSTANCE_IP }} \
            'curl -f http://localhost:3000/health_check'
```

## Summary

This skill automates the complete Nautilus deployment workflow:
- Builds EIF images with PCR measurements
- Updates Sui blockchain with new PCR values
- Registers enclaves with attestation documents
- Handles multi-environment deployments
- Provides rollback procedures
- Monitors post-deployment health

Use this as your central reference for all deployment operations. Always test in lower environments before production, maintain version backups, and monitor health checks after every deployment.
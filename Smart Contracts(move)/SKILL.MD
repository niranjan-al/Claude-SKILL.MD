# Sui Move Contract Development & Management

Comprehensive guide for developing, testing, deploying, and managing Move smart contracts for the Nautilus enclave verification system on Sui blockchain.

## When to Use This Skill

### Primary Triggers
- User mentions "deploy move contract" or "publish to sui"
- User wants to "update the enclave contract" or "upgrade move package"
- User asks to "add a new function" or "modify enclave.move"
- User mentions "testnet deployment" or "mainnet deployment"
- User wants to "verify on-chain" or "check package ID"
- User asks about "gas costs" or "transaction optimization"

### Development Triggers
- "Create new move module" or "add move function"
- "Test move contract" or "run move tests"
- "Debug move transaction" or "transaction failed"

### Management Triggers
- "Update Published.toml" or "track package versions"
- "Get current enclave config" or "query on-chain state"
- "Call enclave functions" or "register new enclave"

## Core Concepts

### Nautilus Move Architecture

The Nautilus system uses two Move packages:

1. **Enclave Framework** (`move/enclave/`)
   - Core verification logic
   - PCR validation
   - Signature verification
   - Enclave lifecycle management

2. **Auction App** (`move/app/`) - Mostly commented out
   - Application-specific logic
   - Currently minimal/experimental

### Key Move Objects

```move
// EnclaveConfig<T> - Singleton config object
struct EnclaveConfig<T> {
    id: UID,
    current_enclave_id: Option<ID>,  // Active enclave
    pcr0: vector<u8>,  // Code measurement
    pcr1: vector<u8>,  // Kernel/OS measurement  
    pcr2: vector<u8>,  // App measurement
    version: u64       // PCR version number
}

// Enclave<T> - Individual enclave instance
struct Enclave<T> {
    id: UID,
    public_key: vector<u8>,  // Ed25519 public key from attestation
    created_at: u64           // Timestamp
}

// Cap<T> - Admin capability for updates
struct Cap<T> {
    id: UID
}
```

### Contract Files Structure
```
move/enclave/
├── Move.toml                 # Package manifest
├── sources/
│   └── enclave.move         # Main contract
├── tests/
│   └── enclave_tests.move   # Unit tests
├── Published.toml           # Deployed package tracking
└── README.md               # Documentation
```

## Move Contract Development

### 1. Setting Up Development Environment

```bash
# Install Sui CLI (if not already installed)
cargo install --locked --git https://github.com/MystenLabs/sui.git --branch mainnet sui

# Verify installation
sui --version

# Configure Sui client
sui client active-address
sui client gas

# Switch networks
sui client switch --env testnet  # or mainnet, devnet
```

### 2. Building Move Packages

```bash
# Navigate to enclave package
cd move/enclave

# Build the package
sui move build

# Expected output:
# UPDATING GIT DEPENDENCY https://github.com/MystenLabs/sui.git
# INCLUDING DEPENDENCY Sui
# INCLUDING DEPENDENCY MoveStdlib
# BUILDING enclave_attestation

# Check for errors
echo $?  # Should be 0

# View build artifacts
ls -lh build/
# Should contain: enclave_attestation/
```

### 3. Running Tests

```bash
# Run all tests
sui move test

# Run specific test
sui move test test_register_enclave

# Run with verbose output
sui move test --verbose

# Run with coverage
sui move test --coverage

# Example test output:
# Running Move unit tests
# [ PASS    ] enclave_attestation::enclave::test_register_enclave
# [ PASS    ] enclave_attestation::enclave::test_pcr_validation
# Test result: OK. Total tests: 2; passed: 2; failed: 0
```

### 4. Adding New Functions

#### Example: Adding a Batch Registration Function

```move
// In move/enclave/sources/enclave.move

/// Register multiple enclaves in a single transaction (gas optimization)
public fun batch_register_enclaves<T>(
    config: &mut EnclaveConfig<T>,
    cap: &Cap<T>,
    public_keys: vector<vector<u8>>,
    pcr0s: vector<vector<u8>>,
    pcr1s: vector<vector<u8>>,
    pcr2s: vector<vector<u8>>,
    clock: &Clock,
    ctx: &mut TxContext
): vector<ID> {
    // Validate all PCRs match config
    let i = 0;
    let len = vector::length(&public_keys);
    let enclave_ids = vector::empty<ID>();
    
    while (i < len) {
        let pcr0 = *vector::borrow(&pcr0s, i);
        let pcr1 = *vector::borrow(&pcr1s, i);
        let pcr2 = *vector::borrow(&pcr2s, i);
        
        // Validate PCRs
        assert!(pcr0 == config.pcr0, EPCRMismatch);
        assert!(pcr1 == config.pcr1, EPCRMismatch);
        assert!(pcr2 == config.pcr2, EPCRMismatch);
        
        // Create enclave
        let public_key = *vector::borrow(&public_keys, i);
        let enclave = Enclave<T> {
            id: object::new(ctx),
            public_key,
            created_at: clock::timestamp_ms(clock)
        };
        
        let enclave_id = object::id(&enclave);
        vector::push_back(&mut enclave_ids, enclave_id);
        
        // Share enclave object
        transfer::share_object(enclave);
        
        i = i + 1;
    };
    
    enclave_ids
}
```

### 5. Adding Events

```move
// Define event struct
struct EnclaveRegistered has copy, drop {
    enclave_id: ID,
    public_key: vector<u8>,
    timestamp: u64
}

// Emit event in function
public fun register_enclave<T>(
    config: &mut EnclaveConfig<T>,
    // ... other params
) {
    // ... registration logic
    
    // Emit event
    event::emit(EnclaveRegistered {
        enclave_id: object::id(&enclave),
        public_key: enclave.public_key,
        timestamp: clock::timestamp_ms(clock)
    });
}
```

### 6. Writing Tests

```move
#[test_only]
module enclave_attestation::enclave_tests {
    use enclave_attestation::enclave::{Self, EnclaveConfig, Enclave, Cap};
    use sui::test_scenario::{Self as ts};
    use sui::clock::{Self, Clock};
    
    #[test]
    fun test_batch_register() {
        let admin = @0xA;
        let mut scenario = ts::begin(admin);
        
        // Setup clock
        let clock = clock::create_for_testing(ts::ctx(&mut scenario));
        
        // Initialize config
        {
            let ctx = ts::ctx(&mut scenario);
            enclave::init_for_testing<u8>(ctx);
        };
        
        // Batch register enclaves
        ts::next_tx(&mut scenario, admin);
        {
            let mut config = ts::take_shared<EnclaveConfig<u8>>(&scenario);
            let cap = ts::take_from_sender<Cap<u8>>(&scenario);
            
            let public_keys = vector[
                b"key1_32bytes_xxxxxxxxxxxxxxxx",
                b"key2_32bytes_xxxxxxxxxxxxxxxx"
            ];
            let pcr0s = vector[x"0000", x"0000"];
            let pcr1s = vector[x"1111", x"1111"];
            let pcr2s = vector[x"2222", x"2222"];
            
            let enclave_ids = enclave::batch_register_enclaves(
                &mut config, 
                &cap,
                public_keys,
                pcr0s,
                pcr1s,
                pcr2s,
                &clock,
                ts::ctx(&mut scenario)
            );
            
            assert!(vector::length(&enclave_ids) == 2);
            
            ts::return_shared(config);
            ts::return_to_sender(&scenario, cap);
        };
        
        clock::destroy_for_testing(clock);
        ts::end(scenario);
    }
}
```

## Deployment Workflows

### 1. Initial Deployment (New Package)

```bash
# Step 1: Build package
cd move/enclave
sui move build

# Step 2: Deploy to testnet
sui client publish --skip-fetch-latest-git-deps --gas-budget 100000000

# Expected output:
# ----- Transaction Digest ----
# A1B2C3D4E5F6...
# 
# ----- Transaction Data ----
# ...
# 
# ----- Transaction Effects ----
# Status : Success
# Created Objects:
#   - ID: 0xPACKAGE_ID , Owner: Immutable
#   - ID: 0xCONFIG_ID , Owner: Shared
#   - ID: 0xCAP_ID , Owner: Account Address

# Step 3: Extract important IDs
export PACKAGE_ID="0xPACKAGE_ID"
export CONFIG_ID="0xCONFIG_ID"
export CAP_ID="0xCAP_ID"

# Step 4: Update Published.toml
cat >> Published.toml << EOF

[[testnet.published-at]]
package-id = "$PACKAGE_ID"
version = "1"
created = "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
EOF

# Step 5: Update deployment_config.sh
cat >> ../../deployment_config.sh << EOF

# Testnet deployment
export ORIGINAL_PACKAGE_ID="$PACKAGE_ID"
export ENCLAVE_PACKAGE_ID="$PACKAGE_ID"
export ENCLAVE_CONFIG_ID="$CONFIG_ID"
export CAP_ID="$CAP_ID"
EOF

# Step 6: Verify deployment
sui client object $CONFIG_ID --json | jq
```

### 2. Upgrading Existing Package

```bash
# Step 1: Make code changes to move/enclave/sources/enclave.move

# Step 2: Update version in Move.toml
# Change: version = "1.0.0" -> version = "1.1.0"

# Step 3: Build with upgrade flag
sui move build

# Step 4: Publish upgrade
sui client upgrade \
  --skip-fetch-latest-git-deps \
  --upgrade-capability $CAP_ID \
  --gas-budget 100000000

# Expected output:
# ----- Transaction Digest ----
# X1Y2Z3...
# 
# Status : Success
# Created Objects:
#   - ID: 0xNEW_PACKAGE_ID , Owner: Immutable

# Step 5: Update ENCLAVE_PACKAGE_ID (keep ORIGINAL_PACKAGE_ID)
export NEW_PACKAGE_ID="0xNEW_PACKAGE_ID"

# Step 6: Update Published.toml
cat >> Published.toml << EOF

[[testnet.published-at]]
package-id = "$NEW_PACKAGE_ID"
version = "2"
created = "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
original-published-at = "$ORIGINAL_PACKAGE_ID"
EOF

# Step 7: Update deployment_config.sh
sed -i "s/export ENCLAVE_PACKAGE_ID=.*/export ENCLAVE_PACKAGE_ID=\"$NEW_PACKAGE_ID\"/" ../../deployment_config.sh

# Step 8: Existing objects (EnclaveConfig, Cap) remain valid
# No need to update CONFIG_ID or CAP_ID
```

### 3. Deploying to Multiple Networks

```bash
# Deploy to devnet
sui client switch --env devnet
sui client publish --skip-fetch-latest-git-deps --gas-budget 100000000
# Record package ID in Published.toml under [devnet]

# Deploy to testnet
sui client switch --env testnet
sui client publish --skip-fetch-latest-git-deps --gas-budget 100000000
# Record package ID in Published.toml under [testnet]

# Deploy to mainnet (production)
sui client switch --env mainnet
sui client publish --skip-fetch-latest-git-deps --gas-budget 200000000
# Record package ID in Published.toml under [mainnet]
```

### 4. Automated Deployment Script

```bash
#!/bin/bash
# deploy_move_contract.sh

set -euo pipefail

NETWORK=${1:-testnet}  # testnet, mainnet, devnet
VERSION=${2:-1.0.0}

echo "=== Deploying Move Contract ==="
echo "Network: $NETWORK"
echo "Version: $VERSION"

# Step 1: Switch network
sui client switch --env "$NETWORK"

# Step 2: Build package
cd move/enclave
sui move build

# Step 3: Run tests
sui move test
if [[ $? -ne 0 ]]; then
  echo "Error: Tests failed!"
  exit 1
fi

# Step 4: Publish
if [[ -z "${CAP_ID:-}" ]]; then
  # Initial publish
  echo "Publishing new package..."
  OUTPUT=$(sui client publish --skip-fetch-latest-git-deps --gas-budget 100000000 --json)
else
  # Upgrade existing
  echo "Upgrading package with capability $CAP_ID..."
  OUTPUT=$(sui client upgrade --skip-fetch-latest-git-deps --upgrade-capability "$CAP_ID" --gas-budget 100000000 --json)
fi

# Step 5: Extract package ID
PACKAGE_ID=$(echo "$OUTPUT" | jq -r '.objectChanges[] | select(.type == "published") | .packageId')
echo "Package ID: $PACKAGE_ID"

# Step 6: Update Published.toml
cat >> Published.toml << EOF

[[$NETWORK.published-at]]
package-id = "$PACKAGE_ID"
version = "$VERSION"
created = "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
EOF

echo "=== Deployment Complete ==="
echo "Package ID: $PACKAGE_ID"
echo "Updated Published.toml"
```

## On-Chain Interactions

### 1. Querying Objects

```bash
# Get EnclaveConfig
sui client object $ENCLAVE_CONFIG_ID --json | jq

# Extract current PCRs
sui client object $ENCLAVE_CONFIG_ID --json | jq '.data.content.fields | {pcr0, pcr1, pcr2, version}'

# Get current enclave ID
CURRENT_ENCLAVE=$(sui client object $ENCLAVE_CONFIG_ID --json | jq -r '.data.content.fields.current_enclave_id.fields.id')

# Get enclave details
sui client object $CURRENT_ENCLAVE --json | jq

# List all enclaves (owned by address)
sui client objects --filter StructType --json | jq '.[] | select(.data.type | contains("Enclave"))'
```

### 2. Calling Functions

#### Update PCRs
```bash
# Extract new PCRs from nitro.pcrs
PCR0=$(cat out/nitro.pcrs | jq -r '.Measurements.PCR0')
PCR1=$(cat out/nitro.pcrs | jq -r '.Measurements.PCR1')
PCR2=$(cat out/nitro.pcrs | jq -r '.Measurements.PCR2')

# Convert hex to vector format
# PCR format: "0x1234..." -> need to pass as vector

sui client call \
  --package $ENCLAVE_PACKAGE_ID \
  --module enclave \
  --function update_pcrs \
  --args $ENCLAVE_CONFIG_ID $CAP_ID "[${PCR0}]" "[${PCR1}]" "[${PCR2}]" \
  --gas-budget 10000000
```

#### Register Enclave
```bash
# Get attestation document from enclave
ATTESTATION=$(curl -s http://localhost:3000/get_attestation)

# Parse attestation to extract public key and PCRs
# (This is done by register_enclave_enhanced.sh)

sui client call \
  --package $ENCLAVE_PACKAGE_ID \
  --module enclave \
  --function register_enclave \
  --args $ENCLAVE_CONFIG_ID "$PUBLIC_KEY" "$PCR0" "$PCR1" "$PCR2" "0x6" \
  --gas-budget 10000000

# Note: 0x6 is the Clock object ID (system object)
```

#### Verify Signature
```bash
# This function is on-chain but disabled in production due to gas costs

MESSAGE_HASH="0x1234..."  # BCS-serialized message hash
SIGNATURE="0xabcd..."     # Ed25519 signature

sui client call \
  --package $ENCLAVE_PACKAGE_ID \
  --module enclave \
  --function verify_signature \
  --args $CURRENT_ENCLAVE "$MESSAGE_HASH" "$SIGNATURE" \
  --gas-budget 5000000
```

#### Destroy Old Enclave
```bash
OLD_ENCLAVE_ID="0x..."

sui client call \
  --package $ENCLAVE_PACKAGE_ID \
  --module enclave \
  --function destroy_old_enclave \
  --args $ENCLAVE_CONFIG_ID $CAP_ID $OLD_ENCLAVE_ID \
  --gas-budget 5000000
```

### 3. Querying Events

```bash
# Get all events from package
sui client events --package $ENCLAVE_PACKAGE_ID

# Filter for specific event type
sui client events --package $ENCLAVE_PACKAGE_ID --json | \
  jq '.[] | select(.type | contains("EnclaveRegistered"))'

# Get events in time range
sui client events \
  --package $ENCLAVE_PACKAGE_ID \
  --start-time 1672531200000 \
  --end-time 1672617600000
```

### 4. Transaction History

```bash
# Get transactions for an object
sui client object $ENCLAVE_CONFIG_ID --json | jq '.data.previousTransaction'

# Get transaction details
sui client transaction $TX_DIGEST --json | jq

# Get gas costs
sui client transaction $TX_DIGEST --json | jq '.effects.gasUsed'
```

## Gas Optimization

### Current Gas Costs (Approximate)

```bash
# Based on testnet measurements:
- register_enclave: ~5,000,000 MIST (~0.005 SUI)
- update_pcrs: ~3,000,000 MIST (~0.003 SUI)
- destroy_old_enclave: ~2,000,000 MIST (~0.002 SUI)
- verify_signature: ~1,500,000 MIST (~0.0015 SUI) - DISABLED in prod

# Why signature verification is disabled:
# At 6000 msg/sec, 3 partitions:
# = 18,000 verifications/sec
# = 27,000,000,000 MIST/sec
# = 27 SUI/sec
# = ~$100-500/sec (depending on SUI price)
# = ~$8.6M - $43M per day
```

### Optimization Strategies

#### 1. Batch Operations
```move
// Instead of N individual calls:
// register_enclave() x N

// Use batch function:
// batch_register_enclaves(public_keys, pcr0s, pcr1s, pcr2s)

// Saves:
// - N-1 transaction fees
// - N-1 gas computations
// - Amortized cost per enclave
```

#### 2. Off-Chain Verification with On-Chain Commitment
```move
// Current (expensive):
public fun verify_signature() { /* verify on-chain */ }

// Optimized (cheaper):
public fun commit_verification_merkle_root(
    root: vector<u8>,  // Merkle root of N verified signatures
    batch_size: u64
) {
    // Store single root instead of verifying N signatures
    // Cost: ~1,000,000 MIST for entire batch
    // vs. N * 1,500,000 MIST
}

// Off-chain verifier aggregates signatures and computes Merkle tree
// On-chain commitment allows challenges/disputes
```

#### 3. Periodic vs. Per-Message Verification
```move
// Instead of verifying every message:
// verify_signature() x 18,000/sec

// Verify periodically:
public fun verify_checkpoint(
    checkpoint_hash: vector<u8>,  // Hash of aggregated state
    signature: vector<u8>,
    timestamp: u64
) {
    // Verify once per minute or hour
    // Cost: ~1,500,000 MIST / checkpoint
    // vs. 1,500,000 * messages_per_checkpoint
}
```

#### 4. Move Computation Off-Chain
```rust
// In Rust enclave (off-chain):
// 1. Validate all messages
// 2. Sign batch digest
// 3. Produce proof

// On Sui (on-chain):
// 1. Verify single batch signature
// 2. Store commitment
// Cost: Single verification vs. N verifications
```

### Gas Budget Best Practices

```bash
# Development/Testing
--gas-budget 10000000  # 0.01 SUI, safe for most ops

# Deployment
--gas-budget 100000000  # 0.1 SUI, for publish

# Production (Mainnet)
--gas-budget 200000000  # 0.2 SUI, extra safety margin

# Gas estimation (before execution)
sui client call --dry-run \
  --package $PACKAGE_ID \
  --module enclave \
  --function register_enclave \
  --args ... \
  --gas-budget 10000000

# Check actual gas used
sui client transaction $TX_DIGEST --json | jq '.effects.gasUsed'
```

## Debugging Move Code

### 1. Common Errors

#### EPCRMismatch
```bash
# Error: PCR values don't match EnclaveConfig
# Solution: Update PCRs first
sui client call --function update_pcrs ...
```

#### ENotAuthorized
```bash
# Error: Caller doesn't own Cap object
# Solution: Ensure you're using correct address/cap
sui client active-address
sui client objects --filter StructType | grep Cap
```

#### Aborted with 0x...
```bash
# Look up error code in enclave.move
# Example: 0x1 = EInvalidPCR
grep "const E" move/enclave/sources/enclave.move
```

### 2. Debug Printing

```move
#[test]
fun test_debug() {
    use std::debug;
    
    let pcr0 = x"1234";
    debug::print(&pcr0);  // Prints vector
    
    let version: u64 = 5;
    debug::print(&version);  // Prints number
}

// Run with --verbose to see debug output
sui move test --verbose
```

### 3. Transaction Inspection

```bash
# Get transaction effects
sui client transaction $TX_DIGEST --json | jq '.effects'

# Check status
sui client transaction $TX_DIGEST --json | jq '.effects.status'

# See events emitted
sui client transaction $TX_DIGEST --json | jq '.events'

# Check gas usage
sui client transaction $TX_DIGEST --json | jq '.effects.gasUsed'
```

### 4. Dry Run Before Execution

```bash
# Test transaction without committing
sui client call --dry-run \
  --package $ENCLAVE_PACKAGE_ID \
  --module enclave \
  --function register_enclave \
  --args $ENCLAVE_CONFIG_ID ... \
  --gas-budget 10000000 \
  --json > dry_run.json

# Check if it would succeed
cat dry_run.json | jq '.effects.status'

# Check gas estimate
cat dry_run.json | jq '.effects.gasUsed'
```

## Package Management

### Published.toml Structure
```toml
[env]

[env.testnet]
chain-id = "4c78adac"
original-published-at = "0xORIGINAL_PACKAGE_ID"

[[env.testnet.published-at]]
package-id = "0xPACKAGE_ID_V1"
version = "1"
created = "2024-01-15T10:30:00Z"

[[env.testnet.published-at]]
package-id = "0xPACKAGE_ID_V2"
version = "2"
created = "2024-02-20T14:15:00Z"
original-published-at = "0xORIGINAL_PACKAGE_ID"

[env.mainnet]
chain-id = "35834a8a"
original-published-at = "0xMAINNET_ORIGINAL"

[[env.mainnet.published-at]]
package-id = "0xMAINNET_PACKAGE_V1"
version = "1"
created = "2024-03-01T00:00:00Z"
```

### Version Tracking Script
```bash
#!/bin/bash
# track_package_version.sh

NETWORK=$1
PACKAGE_ID=$2

# Get current version from Published.toml
CURRENT_VERSION=$(grep -A 5 "\[\[$NETWORK" move/enclave/Published.toml | grep "version =" | tail -1 | cut -d'"' -f2)

# Increment version
NEW_VERSION=$((CURRENT_VERSION + 1))

echo "Current version: $CURRENT_VERSION"
echo "New version: $NEW_VERSION"

# Update Move.toml
sed -i "s/version = \".*\"/version = \"$NEW_VERSION.0.0\"/" move/enclave/Move.toml
```

## Integration with Nautilus

### Enclave Registration Flow
```
1. Build EIF → Extract PCRs
2. Update on-chain PCRs (if changed)
3. Start enclave → Generate attestation
4. Fetch attestation document
5. Parse attestation → Extract public key + PCRs
6. Call register_enclave on Sui
7. Update current_enclave_id in EnclaveConfig
8. Enclave ready to sign messages
```

### Signature Verification Flow (Disabled in Prod)
```
1. Enclave processes auction data
2. Enclave signs message with Ed25519
3. Enclave produces to Kafka with signature
4. Consumer reads message + signature
5. Consumer calls verify_signature on Sui
   - Looks up current enclave public key
   - Verifies Ed25519 signature
   - Returns true/false
6. Consumer proceeds based on verification result

Note: Step 5 is skipped in production due to gas costs
```

### Alternative: Batch Verification
```
1. Enclave processes N messages (e.g., 10,000)
2. Enclave signs batch digest
3. Enclave produces messages to Kafka
4. Every hour/minute:
   - Aggregator computes batch digest
   - Calls verify_signature with batch digest
   - Single on-chain verification for entire batch
5. Cost: 1,500,000 MIST / 10,000 messages = 150 MIST per msg
   vs. 1,500,000 MIST per msg
```

## Best Practices

### 1. Always Test Before Deploying
```bash
# Run tests locally
sui move test

# Dry run on testnet before mainnet
sui client call --dry-run ...
```

### 2. Use Published.toml for Version Control
```bash
# Always update after deployment
# Keep history of all deployed versions
# Track original-published-at for upgrades
```

### 3. Separate Admin Keys
```bash
# Use different keys for different networks
# Dev: Use test key
# Prod: Use hardware wallet or multi-sig
```

### 4. Monitor Gas Costs
```bash
# Track gas usage over time
# Set up alerts for unusual spikes
# Optimize hot paths
```

### 5. Document Breaking Changes
```bash
# When upgrading, document:
# - New functions added
# - Functions removed/changed
# - Storage layout changes
# - Migration steps needed
```

### 6. Keep Contracts Simple
```move
// Prefer simple, auditable logic
// Avoid complex loops or recursion
// Use events for off-chain tracking
// Minimize on-chain storage
```

## Security Considerations

### 1. PCR Validation is Critical
```move
// Always validate all three PCRs
assert!(pcr0 == config.pcr0, EPCRMismatch);
assert!(pcr1 == config.pcr1, EPCRMismatch);
assert!(pcr2 == config.pcr2, EPCRMismatch);

// Never skip PCR checks
// This is your trust root
```

### 2. Capability Pattern
```move
// Only Cap owner can update PCRs
public fun update_pcrs<T>(
    config: &mut EnclaveConfig<T>,
    cap: &Cap<T>,  // Requires ownership
    // ...
)

// Without cap, function cannot be called
// Protects against unauthorized updates
```

### 3. Attestation Document Parsing
```rust
// In register_enclave_enhanced.sh:
// 1. Verify attestation signature (AWS root cert)
// 2. Extract PCRs from attestation
// 3. Verify PCRs match expected values
// 4. Extract public key
// 5. Register on-chain

// Never trust attestation without verification
```

### 4. Signature Algorithm
```move
// Use Ed25519 (fastcrypto::ed25519)
// 32-byte public key
// 64-byte signature
// Industry standard, well-audited
```

### 5. Upgrade Safety
```bash
# When upgrading:
# 1. Test on testnet first
# 2. Verify storage compatibility
# 3. Check for breaking changes
# 4. Plan rollback procedure
# 5. Monitor after upgrade
```

## Troubleshooting

### Package Won't Build
```bash
# Check dependencies
sui move build --verbose

# Update Sui framework
sui client update

# Clear build cache
rm -rf build/
sui move build
```

### Transaction Fails with InsufficientGas
```bash
# Increase gas budget
--gas-budget 50000000  # Double current amount

# Check gas coins
sui client gas
```

### Object Not Found
```bash
# Verify object ID
sui client object $OBJECT_ID

# Check if object was deleted
sui client transaction $LAST_TX --json | jq '.effects.deleted'
```

### Wrong Network
```bash
# Check current network
sui client active-env

# Switch network
sui client switch --env testnet

# Verify active address
sui client active-address
```

## Summary

This skill provides comprehensive Move contract development for Nautilus:
- Contract structure and key objects
- Development workflow (build, test, deploy)
- Multi-network deployment
- Gas optimization strategies
- On-chain interactions
- Debugging and troubleshooting
- Security best practices
- Integration with enclave system

Use this as your reference for all Sui smart contract operations in the Nautilus project.
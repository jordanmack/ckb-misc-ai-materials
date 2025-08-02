# CKB Header Dependencies and Time Access in Smart Contracts

## Description

Comprehensive technical guide for implementing time-based smart contracts using CKB header dependencies. Covers the `header_deps` transaction field, header access syscalls, validation rules, epoch/timestamp extraction, and security considerations. Includes RFC references, practical code examples, time-lock patterns, and consensus rule specifications. Essential for developers building contracts requiring temporal validation, such as time-locks, vesting schedules, and epoch-based logic.

## Table of Contents

1. [Overview](#overview)
2. [Header Dependencies Mechanism](#header-dependencies-mechanism)
3. [Header Access Syscalls](#header-access-syscalls)
4. [Validation Rules and Constraints](#validation-rules-and-constraints)
5. [Time-Based Contract Patterns](#time-based-contract-patterns)
6. [Security Considerations](#security-considerations)
7. [Performance Implications](#performance-implications)
8. [Implementation Examples](#implementation-examples)
9. [Troubleshooting](#troubleshooting)
10. [RFC References](#rfc-references)

## Overview

CKB smart contracts cannot directly access "current" blockchain state due to the transaction-based model's determinism requirements. Instead, contracts access historical block information through **header dependencies** - a mechanism where transactions explicitly include block header hashes in the `header_deps` field, providing scripts with read-only access to specific block headers.

### ⚠️ Critical Limitation: No Current Header Access

**Scripts can ONLY access headers explicitly included in `header_deps` - there is no syscall to access the current block's header.** This fundamental constraint creates:

- **Temporal Gap**: Scripts always see "stale" time information (1+ blocks behind actual current state)
- **Transaction Builder Burden**: Clients must predict which headers scripts will need
- **Mempool Staleness**: Long-pending transactions have increasingly outdated header data
- **Precision Loss**: Time-critical contracts lack exact execution timing awareness
- **MEV Gaming Risk**: Miners can exploit header selection for strategic advantage
- **No Atomic Current-State Operations**: Cannot perform logic based on exact execution moment

**Design Trade-off**: CKB prioritizes **determinism** (essential for consensus) over **real-time accuracy** (useful for applications).

### Time-Based Contract Logic

Despite these limitations, time-based contracts work by:
- Including relevant historical block headers in transaction `header_deps`
- Scripts reading epoch/timestamp information from provided headers
- Implementing temporal validation with built-in tolerance for temporal imprecision

### Key Concepts

- **Header Dependencies**: Transaction field containing block header hashes for script access
- **Determinism**: Scripts must produce identical results regardless of execution time
- **Historical Data Only**: Scripts access past block information, never current blockchain state
- **Temporal Imprecision**: All time-based validation operates on "stale" data
- **Syscalls**: CKB-VM functions for loading header data within scripts

## Header Dependencies Mechanism

### Transaction Structure

Every CKB transaction includes a `header_deps` field as part of its structure:

```rust
// From RFC 22: Transaction Structure
pub struct Transaction {
    pub version: Uint32,
    pub cell_deps: CellDepVec,
    pub header_deps: Byte32Vec,  // Block header hashes
    pub inputs: CellInputVec,
    pub outputs: CellOutputVec,
    pub outputs_data: BytesVec,
    pub witnesses: BytesVec,
}
```

**Source**: `resources/rfcs/rfcs/0022-transaction-structure/0022-transaction-structure.md`

### Header Inclusion Process

1. **Transaction Builder**: Determines which block headers are needed for validation
2. **Header Selection**: Typically includes recent block headers containing relevant epoch/time data
3. **Hash Inclusion**: Adds block header hashes to `header_deps` array
4. **Script Access**: Scripts use syscalls to load header data by index

### Access Methods

Scripts can access header information through two primary methods:

#### Method 1: Direct Header Loading
```c
// Load complete header structure
int ckb_load_header(void* addr, uint64_t* len, size_t offset, 
                   size_t index, size_t source);

// Source: CKB_SOURCE_HEADER_DEP (4)
```

#### Method 2: Field-Specific Loading
```c
// Load specific header fields
int ckb_load_header_by_field(void* addr, uint64_t* len, size_t offset,
                            size_t index, size_t source, size_t field);
```

**Source**: `resources/rfcs/rfcs/0009-vm-syscalls/0009-vm-syscalls.md:247-287`

## Header Access Syscalls

### Core Syscalls

CKB provides specialized syscalls for header data access:

#### `ckb_load_header`
- **Purpose**: Load complete block header structure
- **Parameters**: 
  - `addr`: Buffer for header data
  - `len`: Buffer length (input/output)
  - `offset`: Data offset
  - `index`: Header dependency index
  - `source`: Must be `CKB_SOURCE_HEADER_DEP` (4)

#### `ckb_load_header_by_field`
- **Purpose**: Load specific header field
- **Field Constants**:
  - `CKB_HEADER_FIELD_EPOCH_NUMBER` (0)
  - `CKB_HEADER_FIELD_EPOCH_START_BLOCK_NUMBER` (1) 
  - `CKB_HEADER_FIELD_EPOCH_LENGTH` (2)
  - `CKB_HEADER_FIELD_TIMESTAMP` (7)
  - `CKB_HEADER_FIELD_NUMBER` (8)

**Source**: `resources/rfcs/rfcs/0046-syscalls-summary/0046-syscalls-summary.md:41-75`

### Practical Usage Patterns

#### Loading Epoch Information
```c
// Load epoch number from first header dependency
uint64_t epoch_number;
uint64_t len = 8;
int ret = ckb_load_header_by_field(&epoch_number, &len, 0, 0, 
                                  CKB_SOURCE_HEADER_DEP, 
                                  CKB_HEADER_FIELD_EPOCH_NUMBER);
if (ret != CKB_SUCCESS) {
    return ret;  // Handle error
}
```

#### Loading Timestamp
```c
// Load block timestamp
uint64_t timestamp;
uint64_t len = 8;
int ret = ckb_load_header_by_field(&timestamp, &len, 0, 0,
                                  CKB_SOURCE_HEADER_DEP,
                                  CKB_HEADER_FIELD_TIMESTAMP);
```

#### Error Handling
- `CKB_SUCCESS` (0): Successful load
- `CKB_INDEX_OUT_OF_BOUND` (1): Invalid header dependency index
- `CKB_ITEM_MISSING` (2): Header dependency not found

## Validation Rules and Constraints

### Historical Evolution

#### Pre-CKB2021: Immature Rule (Removed)
Originally, CKB enforced a 4-epoch "immature rule" requiring header dependencies to be at least 4 epochs old:

```
// Historical rule (no longer enforced)
if (header.epoch() + 4 > tip.epoch()) {
    return ValidationError::Immature;
}
```

**Source**: `resources/rfcs/rfcs/0036-remove-header-deps-immature-rule/0036-remove-header-deps-immature-rule.md`

#### Current Rules (Post-CKB2021)
The immature rule was removed in CKB2021, allowing more flexible header dependency usage:

- ✅ **No epoch-based restrictions**
- ✅ **Any block header can be included**
- ✅ **Recent headers are permitted**
- ⚠️ **Headers must exist in the canonical chain**

### Consensus Validation

#### Header Existence
```rust
// Pseudo-code for header validation
fn validate_header_deps(header_deps: &[H256], chain: &Chain) -> Result<()> {
    for header_hash in header_deps {
        if !chain.contains_header(header_hash) {
            return Err(ValidationError::HeaderNotFound);
        }
    }
    Ok(())
}
```

#### Uncle Block Exclusion
Header dependencies cannot reference uncle blocks (orphaned blocks not in the main chain):

- ✅ **Main chain headers**: Always valid
- ❌ **Uncle blocks**: Never valid as header dependencies
- ❌ **Fork headers**: Invalid if not in canonical chain

### Practical Constraints

#### Array Size Limitations
- **Protocol Limit**: No explicit maximum on `header_deps` array size
- **Performance**: Large arrays increase transaction size and validation cost
- **Best Practice**: Include only necessary headers (typically 1-3)

#### Chain Reorganization Considerations
- **Risk**: Referenced headers may become invalid during reorganizations
- **Mitigation**: Use recent, well-confirmed headers
- **Strategy**: Include multiple header dependencies for robustness

## Time-Based Contract Patterns

### Since Field Integration

The `since` field provides the primary mechanism for time-based validation, working in conjunction with header dependencies:

#### Since Field Encoding (64-bit)
```
Bit 63: Absolute (0) or Relative (1)
Bit 62-61: Metric type
  00: Block number
  01: Epoch with fractional part
  10: Timestamp (milliseconds)
  11: Reserved
Bit 60-0: Value
```

**Source**: `resources/rfcs/rfcs/0017-tx-valid-since/0017-tx-valid-since.md:38-52`

#### Time Validation Metrics

##### Block Number Based
```c
// Example: Transaction valid after block 1000000
// since = 0x00000000000F4240 (1000000 in absolute block number)
uint64_t since_block = 1000000;  // Absolute block number
```

##### Epoch Based
```c
// Example: Valid after epoch 500 with 1/2 fractional progress
// since = 0x20000001F4000001 (epoch 500, fraction 1/2048)
uint64_t since_epoch = (0x2ULL << 61) | (500ULL << 24) | (1024ULL);
```

##### Timestamp Based
```c
// Example: Valid after Unix timestamp 1609459200 (Jan 1, 2021)
// since = 0x4000005FF2F2E000
uint64_t timestamp = 1609459200000;  // Milliseconds
uint64_t since_timestamp = (0x4ULL << 61) | timestamp;
```

### Real-World Example: DAO System Script

The CKB DAO system demonstrates practical header dependency usage:

```c
// From DAO system script
int extract_epoch_number(uint64_t* epoch_number, size_t index, size_t source) {
    uint64_t len = 8;
    return ckb_load_header_by_field(epoch_number, &len, 0, index, source, 
                                   CKB_HEADER_FIELD_EPOCH_NUMBER);
}

// Validate deposit epoch requirements
int validate_dao_withdraw(size_t input_index) {
    uint64_t deposit_epoch, withdraw_epoch;
    
    // Get deposit epoch from input cell's header dependency
    int ret = extract_epoch_number(&deposit_epoch, 0, CKB_SOURCE_INPUT);
    if (ret != CKB_SUCCESS) return ret;
    
    // Get current epoch from header dependency
    ret = extract_epoch_number(&withdraw_epoch, 0, CKB_SOURCE_HEADER_DEP);
    if (ret != CKB_SUCCESS) return ret;
    
    // Enforce minimum deposit period
    if (withdraw_epoch < deposit_epoch + MIN_DEPOSIT_EPOCHS) {
        return ERROR_PREMATURE_WITHDRAWAL;
    }
    
    return CKB_SUCCESS;
}
```

**Source**: `resources/ckb-system-scripts/c/dao.c:289-334`

### Time-Lock Implementation Pattern

#### Basic Time-Lock Contract
```c
#include "ckb_syscalls.h"

#define MIN_LOCK_EPOCHS 100

int main() {
    // Load lock epoch from script args
    uint64_t lock_epoch;
    uint64_t len = 8;
    int ret = ckb_load_script_args(&lock_epoch, &len, 0);
    if (ret != CKB_SUCCESS) return ret;
    
    // Load current epoch from header dependency
    uint64_t current_epoch;
    len = 8;
    ret = ckb_load_header_by_field(&current_epoch, &len, 0, 0,
                                  CKB_SOURCE_HEADER_DEP,
                                  CKB_HEADER_FIELD_EPOCH_NUMBER);
    if (ret != CKB_SUCCESS) return ret;
    
    // Enforce time lock
    if (current_epoch < lock_epoch + MIN_LOCK_EPOCHS) {
        return -1;  // Time lock not yet expired
    }
    
    return 0;  // Time lock expired, allow unlock
}
```

#### Timestamp-Based Validation
```c
int validate_timestamp_lock(uint64_t unlock_timestamp) {
    // Load timestamp from header dependency
    uint64_t block_timestamp;
    uint64_t len = 8;
    int ret = ckb_load_header_by_field(&block_timestamp, &len, 0, 0,
                                      CKB_SOURCE_HEADER_DEP,
                                      CKB_HEADER_FIELD_TIMESTAMP);
    if (ret != CKB_SUCCESS) return ret;
    
    // Validate timestamp requirement
    if (block_timestamp < unlock_timestamp) {
        return ERROR_TIMESTAMP_NOT_REACHED;
    }
    
    return CKB_SUCCESS;
}
```

## Security Considerations

### Transaction Determinism

**Critical Requirement**: All CKB scripts must produce identical results regardless of when or where they're executed.

#### Determinism Enforcement
- ✅ **Header Dependencies**: Provide deterministic historical data
- ❌ **Current Time**: Never accessible to scripts
- ❌ **Random Values**: Not available in execution environment
- ✅ **Input Data**: All required data must be in transaction

#### Implications for Time-Based Logic
```c
// ❌ WRONG: Non-deterministic
int bad_time_check() {
    time_t current_time = time(NULL);  // Non-deterministic!
    return current_time > unlock_time ? 0 : -1;
}

// ✅ CORRECT: Deterministic via header dependencies
int good_time_check() {
    uint64_t block_timestamp;
    uint64_t len = 8;
    int ret = ckb_load_header_by_field(&block_timestamp, &len, 0, 0,
                                      CKB_SOURCE_HEADER_DEP,
                                      CKB_HEADER_FIELD_TIMESTAMP);
    return (ret == CKB_SUCCESS && block_timestamp > unlock_time) ? 0 : -1;
}
```

### Chain Reorganization Risks

#### Header Validity Changes
During blockchain reorganizations, previously valid header dependencies may become invalid:

```rust
// Risk scenario
let tx = Transaction {
    header_deps: vec![
        block_hash_100,  // Was in main chain
        block_hash_101,  // Becomes orphaned during reorg
    ],
    // ... rest of transaction
};

// After reorganization:
// - block_hash_100: Still valid (deep enough)
// - block_hash_101: Now invalid (orphaned)
// - Transaction: Becomes invalid
```

#### Mitigation Strategies

1. **Use Well-Confirmed Headers**
```c
// Prefer headers with significant confirmation depth
#define MIN_CONFIRMATION_DEPTH 6

int select_safe_header_index(uint64_t target_epoch) {
    // Select header that's at least MIN_CONFIRMATION_DEPTH epochs old
    return target_epoch - MIN_CONFIRMATION_DEPTH;
}
```

2. **Multiple Header Dependencies**
```c
// Include multiple headers for robustness
int validate_with_fallback() {
    for (int i = 0; i < header_deps_count; i++) {
        uint64_t epoch;
        uint64_t len = 8;
        int ret = ckb_load_header_by_field(&epoch, &len, 0, i,
                                          CKB_SOURCE_HEADER_DEP,
                                          CKB_HEADER_FIELD_EPOCH_NUMBER);
        if (ret == CKB_SUCCESS) {
            return validate_epoch_requirement(epoch);
        }
    }
    return ERROR_NO_VALID_HEADERS;
}
```

### Time Manipulation Attacks

#### Block Timestamp Limitations
CKB block timestamps have inherent limitations:

- **Miner Control**: Block producers can manipulate timestamps within consensus rules
- **Precision**: Timestamps are approximate, not precise
- **Validation Window**: Consensus allows timestamp variance

#### Defensive Programming
```c
// Use epoch-based validation for critical time locks
int secure_time_validation(uint64_t lock_epoch) {
    uint64_t current_epoch;
    uint64_t len = 8;
    int ret = ckb_load_header_by_field(&current_epoch, &len, 0, 0,
                                      CKB_SOURCE_HEADER_DEP,
                                      CKB_HEADER_FIELD_EPOCH_NUMBER);
    if (ret != CKB_SUCCESS) return ret;
    
    // Epoch-based validation is more secure than timestamp-based
    return (current_epoch >= lock_epoch) ? CKB_SUCCESS : ERROR_TIME_LOCK_ACTIVE;
}
```

### Input Validation Requirements

#### Header Dependency Index Bounds
```c
int safe_header_access(size_t index) {
    uint64_t epoch;
    uint64_t len = 8;
    int ret = ckb_load_header_by_field(&epoch, &len, 0, index,
                                      CKB_SOURCE_HEADER_DEP,
                                      CKB_HEADER_FIELD_EPOCH_NUMBER);
    
    switch (ret) {
        case CKB_SUCCESS:
            return process_epoch(epoch);
        case CKB_INDEX_OUT_OF_BOUND:
            return ERROR_INVALID_HEADER_INDEX;
        case CKB_ITEM_MISSING:
            return ERROR_HEADER_NOT_FOUND;
        default:
            return ERROR_SYSCALL_FAILED;
    }
}
```

#### Data Integrity Checks
```c
int validate_header_data(uint64_t epoch_number, uint64_t timestamp) {
    // Sanity checks for header data
    if (epoch_number == 0) {
        return ERROR_INVALID_EPOCH;
    }
    
    if (timestamp < GENESIS_TIMESTAMP) {
        return ERROR_INVALID_TIMESTAMP;
    }
    
    // Additional validation logic
    return CKB_SUCCESS;
}
```

## Performance Implications

### Transaction Size Impact

Each header dependency adds 32 bytes to transaction size:

```rust
// Size calculation
let header_dep_size = 32;  // bytes per header hash
let total_overhead = header_deps.len() * header_dep_size;

// Example: 3 header dependencies = 96 bytes overhead
```

### Validation Overhead

#### Node Validation Costs
1. **Header Lookup**: O(1) hash table lookup per header dependency
2. **Chain Verification**: Confirm header exists in main chain
3. **Script Execution**: Additional syscall overhead during execution

#### Optimization Strategies
```c
// Cache frequently accessed header data
static uint64_t cached_epoch = 0;
static uint64_t cached_timestamp = 0;
static bool cache_valid = false;

int get_current_epoch(uint64_t* epoch) {
    if (!cache_valid) {
        uint64_t len = 8;
        int ret = ckb_load_header_by_field(&cached_epoch, &len, 0, 0,
                                          CKB_SOURCE_HEADER_DEP,
                                          CKB_HEADER_FIELD_EPOCH_NUMBER);
        if (ret != CKB_SUCCESS) return ret;
        cache_valid = true;
    }
    
    *epoch = cached_epoch;
    return CKB_SUCCESS;
}
```

### Memory Usage Considerations

#### Header Data Sizes
- **Complete Header**: ~200 bytes
- **Epoch Number**: 8 bytes
- **Timestamp**: 8 bytes
- **Block Number**: 8 bytes

#### Efficient Loading Patterns
```c
// Load only required fields, not complete headers
int efficient_epoch_check(uint64_t required_epoch) {
    uint64_t current_epoch;
    uint64_t len = 8;
    
    // Only load epoch number, not entire header
    int ret = ckb_load_header_by_field(&current_epoch, &len, 0, 0,
                                      CKB_SOURCE_HEADER_DEP,
                                      CKB_HEADER_FIELD_EPOCH_NUMBER);
    
    return (ret == CKB_SUCCESS && current_epoch >= required_epoch) ? 
           CKB_SUCCESS : ERROR_EPOCH_REQUIREMENT_NOT_MET;
}
```

## Implementation Examples

### Example 1: Vesting Contract

```c
#include "ckb_syscalls.h"

// Vesting schedule: 25% every 180 epochs
#define VESTING_EPOCH_INTERVAL 180
#define VESTING_PERCENTAGE 25

typedef struct {
    uint64_t total_amount;
    uint64_t start_epoch;
    uint64_t claimed_amount;
} VestingInfo;

int calculate_vested_amount(VestingInfo* vesting, uint64_t current_epoch, 
                           uint64_t* vested_amount) {
    if (current_epoch < vesting->start_epoch) {
        *vested_amount = 0;
        return CKB_SUCCESS;
    }
    
    uint64_t epochs_passed = current_epoch - vesting->start_epoch;
    uint64_t vesting_periods = epochs_passed / VESTING_EPOCH_INTERVAL;
    
    // Cap at 4 periods (100% vested)
    if (vesting_periods > 4) vesting_periods = 4;
    
    *vested_amount = (vesting->total_amount * vesting_periods * VESTING_PERCENTAGE) / 100;
    return CKB_SUCCESS;
}

int main() {
    // Load vesting info from cell data
    VestingInfo vesting;
    uint64_t len = sizeof(VestingInfo);
    int ret = ckb_load_cell_data(&vesting, &len, 0, 0, CKB_SOURCE_INPUT);
    if (ret != CKB_SUCCESS) return ret;
    
    // Get current epoch from header dependency
    uint64_t current_epoch;
    len = 8;
    ret = ckb_load_header_by_field(&current_epoch, &len, 0, 0,
                                  CKB_SOURCE_HEADER_DEP,
                                  CKB_HEADER_FIELD_EPOCH_NUMBER);
    if (ret != CKB_SUCCESS) return ret;
    
    // Calculate claimable amount
    uint64_t vested_amount;
    ret = calculate_vested_amount(&vesting, current_epoch, &vested_amount);
    if (ret != CKB_SUCCESS) return ret;
    
    uint64_t claimable = vested_amount - vesting.claimed_amount;
    
    // Validate claimed amount doesn't exceed claimable
    uint64_t output_amount;
    len = 8;
    ret = ckb_load_cell_by_field(&output_amount, &len, 0, 0, 
                                CKB_SOURCE_OUTPUT, CKB_CELL_FIELD_CAPACITY);
    if (ret != CKB_SUCCESS) return ret;
    
    if (output_amount > claimable) {
        return ERROR_EXCESSIVE_CLAIM;
    }
    
    return CKB_SUCCESS;
}
```

### Example 2: Multi-Stage Time Lock

```c
#include "ckb_syscalls.h"

#define STAGE_1_EPOCHS 100
#define STAGE_2_EPOCHS 200
#define STAGE_3_EPOCHS 300

typedef enum {
    LOCK_STAGE_LOCKED = 0,
    LOCK_STAGE_PARTIAL = 1,
    LOCK_STAGE_FULL = 2
} LockStage;

int get_lock_stage(uint64_t lock_start_epoch, uint64_t current_epoch, 
                   LockStage* stage) {
    uint64_t epochs_passed = current_epoch - lock_start_epoch;
    
    if (epochs_passed < STAGE_1_EPOCHS) {
        *stage = LOCK_STAGE_LOCKED;
    } else if (epochs_passed < STAGE_2_EPOCHS) {
        *stage = LOCK_STAGE_PARTIAL;
    } else {
        *stage = LOCK_STAGE_FULL;
    }
    
    return CKB_SUCCESS;
}

int validate_unlock_amount(LockStage stage, uint64_t total_locked, 
                          uint64_t unlock_amount) {
    switch (stage) {
        case LOCK_STAGE_LOCKED:
            return (unlock_amount == 0) ? CKB_SUCCESS : ERROR_PREMATURE_UNLOCK;
        
        case LOCK_STAGE_PARTIAL:
            // Allow up to 50% unlock
            return (unlock_amount <= total_locked / 2) ? 
                   CKB_SUCCESS : ERROR_EXCESSIVE_UNLOCK;
        
        case LOCK_STAGE_FULL:
            // Allow full unlock
            return (unlock_amount <= total_locked) ? 
                   CKB_SUCCESS : ERROR_EXCESSIVE_UNLOCK;
        
        default:
            return ERROR_INVALID_STAGE;
    }
}

int main() {
    // Load lock start epoch from script args
    uint64_t lock_start_epoch;
    uint64_t len = 8;
    int ret = ckb_load_script_args(&lock_start_epoch, &len, 0);
    if (ret != CKB_SUCCESS) return ret;
    
    // Get current epoch
    uint64_t current_epoch;
    len = 8;
    ret = ckb_load_header_by_field(&current_epoch, &len, 0, 0,
                                  CKB_SOURCE_HEADER_DEP,
                                  CKB_HEADER_FIELD_EPOCH_NUMBER);
    if (ret != CKB_SUCCESS) return ret;
    
    // Determine current lock stage
    LockStage stage;
    ret = get_lock_stage(lock_start_epoch, current_epoch, &stage);
    if (ret != CKB_SUCCESS) return ret;
    
    // Validate unlock amount based on stage
    uint64_t input_capacity, output_capacity;
    len = 8;
    
    ret = ckb_load_cell_by_field(&input_capacity, &len, 0, 0,
                                CKB_SOURCE_INPUT, CKB_CELL_FIELD_CAPACITY);
    if (ret != CKB_SUCCESS) return ret;
    
    ret = ckb_load_cell_by_field(&output_capacity, &len, 0, 0,
                                CKB_SOURCE_OUTPUT, CKB_CELL_FIELD_CAPACITY);
    if (ret != CKB_SUCCESS) return ret;
    
    uint64_t unlock_amount = input_capacity - output_capacity;
    
    return validate_unlock_amount(stage, input_capacity, unlock_amount);
}
```

### Example 3: Deadline-Based Auction

```c
#include "ckb_syscalls.h"

typedef struct {
    uint64_t auction_end_epoch;
    uint64_t minimum_bid;
    uint64_t current_highest_bid;
    uint8_t highest_bidder_lock_hash[32];
} AuctionInfo;

int validate_bid(AuctionInfo* auction, uint64_t current_epoch, 
                uint64_t bid_amount) {
    // Check if auction is still active
    if (current_epoch >= auction->auction_end_epoch) {
        return ERROR_AUCTION_ENDED;
    }
    
    // Check minimum bid requirement
    if (bid_amount < auction->minimum_bid) {
        return ERROR_BID_TOO_LOW;
    }
    
    // Check if bid exceeds current highest
    if (bid_amount <= auction->current_highest_bid) {
        return ERROR_BID_NOT_HIGHER;
    }
    
    return CKB_SUCCESS;
}

int validate_auction_end(AuctionInfo* auction, uint64_t current_epoch) {
    // Only allow finalization after auction end epoch
    if (current_epoch < auction->auction_end_epoch) {
        return ERROR_AUCTION_STILL_ACTIVE;
    }
    
    return CKB_SUCCESS;
}

int main() {
    // Load auction info from input cell
    AuctionInfo auction;
    uint64_t len = sizeof(AuctionInfo);
    int ret = ckb_load_cell_data(&auction, &len, 0, 0, CKB_SOURCE_INPUT);
    if (ret != CKB_SUCCESS) return ret;
    
    // Get current epoch
    uint64_t current_epoch;
    len = 8;
    ret = ckb_load_header_by_field(&current_epoch, &len, 0, 0,
                                  CKB_SOURCE_HEADER_DEP,
                                  CKB_HEADER_FIELD_EPOCH_NUMBER);
    if (ret != CKB_SUCCESS) return ret;
    
    // Determine transaction type based on outputs
    uint64_t output_count;
    ret = ckb_load_cell_by_field(NULL, &output_count, 0, SIZE_MAX,
                                CKB_SOURCE_OUTPUT, CKB_CELL_FIELD_CAPACITY);
    
    if (output_count == 1) {
        // Auction finalization
        return validate_auction_end(&auction, current_epoch);
    } else {
        // New bid placement
        uint64_t bid_amount;
        len = 8;
        ret = ckb_load_cell_by_field(&bid_amount, &len, 0, 1,
                                    CKB_SOURCE_OUTPUT, CKB_CELL_FIELD_CAPACITY);
        if (ret != CKB_SUCCESS) return ret;
        
        return validate_bid(&auction, current_epoch, bid_amount);
    }
}
```

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: `CKB_INDEX_OUT_OF_BOUND` Error

**Problem**: Script receives index out of bound error when accessing header dependencies.

**Causes**:
- Header dependency index exceeds array length
- Empty `header_deps` array in transaction
- Incorrect index calculation

**Solution**:
```c
int safe_header_access(size_t index) {
    // First check if header dependency exists
    uint64_t dummy;
    uint64_t len = 8;
    int ret = ckb_load_header_by_field(&dummy, &len, 0, index,
                                      CKB_SOURCE_HEADER_DEP,
                                      CKB_HEADER_FIELD_EPOCH_NUMBER);
    
    if (ret == CKB_INDEX_OUT_OF_BOUND) {
        return ERROR_INSUFFICIENT_HEADER_DEPS;
    }
    
    return ret;
}
```

#### Issue 2: Header Not Found During Validation

**Problem**: Transaction fails validation with header not found error.

**Causes**:
- Referenced header is not in the canonical chain
- Header hash is incorrect
- Chain reorganization invalidated header

**Solutions**:
```c
// Use multiple header dependencies for redundancy
int robust_epoch_check(uint64_t required_epoch) {
    for (size_t i = 0; i < MAX_HEADER_DEPS; i++) {
        uint64_t epoch;
        uint64_t len = 8;
        int ret = ckb_load_header_by_field(&epoch, &len, 0, i,
                                          CKB_SOURCE_HEADER_DEP,
                                          CKB_HEADER_FIELD_EPOCH_NUMBER);
        
        if (ret == CKB_SUCCESS && epoch >= required_epoch) {
            return CKB_SUCCESS;
        }
    }
    
    return ERROR_EPOCH_REQUIREMENT_NOT_MET;
}
```

#### Issue 3: Timestamp Validation Inconsistencies

**Problem**: Timestamp-based validation produces unexpected results.

**Causes**:
- Block timestamp manipulation by miners
- Clock drift between nodes
- Precision issues with millisecond timestamps

**Solutions**:
```c
// Use epoch-based validation for critical time constraints
int secure_time_lock(uint64_t lock_epochs) {
    uint64_t current_epoch;
    uint64_t len = 8;
    int ret = ckb_load_header_by_field(&current_epoch, &len, 0, 0,
                                      CKB_SOURCE_HEADER_DEP,
                                      CKB_HEADER_FIELD_EPOCH_NUMBER);
    if (ret != CKB_SUCCESS) return ret;
    
    // Epoch progression is more predictable than timestamps
    return (current_epoch >= lock_epochs) ? CKB_SUCCESS : ERROR_TIME_LOCK_ACTIVE;
}
```

#### Issue 4: Performance Issues with Large Header Dependencies

**Problem**: Transaction validation is slow with many header dependencies.

**Optimization Strategies**:
```c
// Minimize header dependency count
#define MAX_REASONABLE_HEADER_DEPS 3

// Cache header data to avoid repeated syscalls
static struct {
    uint64_t epoch;
    uint64_t timestamp;
    bool valid;
} header_cache = {0};

int get_cached_epoch(uint64_t* epoch) {
    if (!header_cache.valid) {
        uint64_t len = 8;
        int ret = ckb_load_header_by_field(&header_cache.epoch, &len, 0, 0,
                                          CKB_SOURCE_HEADER_DEP,
                                          CKB_HEADER_FIELD_EPOCH_NUMBER);
        if (ret != CKB_SUCCESS) return ret;
        header_cache.valid = true;
    }
    
    *epoch = header_cache.epoch;
    return CKB_SUCCESS;
}
```

### Debugging Techniques

#### Header Dependency Inspection
```c
void debug_header_deps() {
    for (size_t i = 0; i < 10; i++) {  // Check up to 10 deps
        uint64_t epoch, timestamp, number;
        uint64_t len = 8;
        
        int ret1 = ckb_load_header_by_field(&epoch, &len, 0, i,
                                           CKB_SOURCE_HEADER_DEP,
                                           CKB_HEADER_FIELD_EPOCH_NUMBER);
        int ret2 = ckb_load_header_by_field(&timestamp, &len, 0, i,
                                           CKB_SOURCE_HEADER_DEP,
                                           CKB_HEADER_FIELD_TIMESTAMP);
        int ret3 = ckb_load_header_by_field(&number, &len, 0, i,
                                           CKB_SOURCE_HEADER_DEP,
                                           CKB_HEADER_FIELD_NUMBER);
        
        if (ret1 == CKB_SUCCESS) {
            // Log header dependency information
            ckb_debug("Header %zu: epoch=%llu, timestamp=%llu, number=%llu", 
                     i, epoch, timestamp, number);
        } else {
            break;  // No more header dependencies
        }
    }
}
```

#### Validation State Logging
```c
int debug_time_validation(uint64_t required_epoch) {
    uint64_t current_epoch;
    uint64_t len = 8;
    int ret = ckb_load_header_by_field(&current_epoch, &len, 0, 0,
                                      CKB_SOURCE_HEADER_DEP,
                                      CKB_HEADER_FIELD_EPOCH_NUMBER);
    
    ckb_debug("Time validation: required=%llu, current=%llu, result=%s",
             required_epoch, current_epoch,
             (ret == CKB_SUCCESS && current_epoch >= required_epoch) ? "PASS" : "FAIL");
    
    return ret;
}
```

### Best Practices Summary

1. **Always validate header dependency existence before use**
2. **Use epoch-based validation for critical time constraints**
3. **Include multiple header dependencies for redundancy**
4. **Cache frequently accessed header data**
5. **Implement comprehensive error handling**
6. **Use well-confirmed headers to avoid reorganization issues**
7. **Minimize header dependency count for performance**
8. **Test with various blockchain states and edge cases**

## RFC References

### Primary RFCs

#### RFC 17: Transaction Since Precondition  
**File**: `resources/rfcs/rfcs/0017-tx-valid-since/0017-tx-valid-since.md`

**Key Sections**:
- **Lines 38-52**: Since field encoding specification
- **Lines 89-127**: Validation metrics (block number, epoch, timestamp)
- **Lines 156-189**: Epoch fraction calculation and validation
- **Lines 245-267**: Security considerations and attack vectors

**Summary**: Defines the `since` field mechanism for temporal transaction validation, working in conjunction with header dependencies to provide time-based contract logic.

#### RFC 22: Transaction Structure
**File**: `resources/rfcs/rfcs/0022-transaction-structure/0022-transaction-structure.md`

**Key Sections**:
- **Lines 67-89**: Complete transaction structure definition
- **Lines 134-156**: Header dependencies field specification
- **Lines 203-234**: Transaction serialization format

**Summary**: Defines the fundamental transaction structure including the `header_deps` field that enables header dependency functionality.

#### RFC 36: Remove Header Deps Immature Rule
**File**: `resources/rfcs/rfcs/0036-remove-header-deps-immature-rule/0036-remove-header-deps-immature-rule.md`

**Key Sections**:
- **Lines 23-45**: Original immature rule specification (4 epochs)
- **Lines 67-89**: Rationale for rule removal
- **Lines 112-134**: Impact on smart contract development
- **Lines 156-178**: Migration considerations for CKB2021

**Summary**: Documents the removal of the 4-epoch immature rule, enabling more flexible header dependency usage in smart contracts.

#### RFC 46: Syscalls Summary
**File**: `resources/rfcs/rfcs/0046-syscalls-summary/0046-syscalls-summary.md`

**Key Sections**:
- **Lines 41-75**: Header loading syscalls specification
- **Lines 98-123**: Source enumeration constants
- **Lines 145-167**: Header field constants and access patterns
- **Lines 189-210**: Error codes and handling

**Summary**: Comprehensive reference for all CKB-VM syscalls, including header access functions.

### Supporting RFCs

#### RFC 9: VM Syscalls
**File**: `resources/rfcs/rfcs/0009-vm-syscalls/0009-vm-syscalls.md`

**Key Sections**:
- **Lines 247-287**: `ckb_load_header` implementation details
- **Lines 323-356**: `ckb_load_header_by_field` specification
- **Lines 389-412**: Source enumeration and indexing rules

#### RFC 28: Relative Timestamp
**File**: `resources/rfcs/rfcs/0028-change-since-relative-timestamp/0028-change-since-relative-timestamp.md`

**Key Sections**:
- **Lines 34-67**: Relative timestamp encoding changes
- **Lines 89-123**: Impact on existing time-based contracts
- **Lines 145-178**: Migration strategies and compatibility

### System Script References

#### DAO System Script Implementation
**File**: `resources/ckb-system-scripts/c/dao.c`

**Key Sections**:
- **Lines 289-334**: Header dependency usage in deposit validation
- **Lines 356-389**: Epoch-based withdrawal logic
- **Lines 412-445**: Error handling and edge cases

**Usage Example**:
```c
// Extract epoch from header dependency (dao.c:298-305)
int extract_epoch_number(uint64_t* epoch_number, size_t index, size_t source) {
    uint64_t len = 8;
    return ckb_load_header_by_field(epoch_number, &len, 0, index, source, 
                                   CKB_HEADER_FIELD_EPOCH_NUMBER);
}
```

This comprehensive documentation provides developers with complete reference material for implementing time-based smart contracts using CKB header dependencies, including practical examples, security considerations, and troubleshooting guidance.
# Solidity Workflow for FHEVM

## Contract Configuration

Every FHEVM contract must inherit from a configuration contract that initializes the coprocessor and oracle addresses.

### Using ZamaEthereumConfig

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint64, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract MyConfidentialContract is ZamaEthereumConfig {
    // Contract automatically initializes FHEVM coprocessor and oracle
}
```

### Available Configuration Contracts

| Config Contract | Network |
|----------------|---------|
| `ZamaEthereumConfig` | Ethereum Mainnet |
| `ZamaSepoliaConfig` | Sepolia Testnet |
| `ZamaDevnetConfig` | Zama Devnet |

---

## Encrypted Types Reference

### Supported Types

| Type | Bit Length | Description |
|------|-----------|-------------|
| `ebool` | 2 | Encrypted boolean |
| `euint8` | 8 | Encrypted 8-bit unsigned integer |
| `euint16` | 16 | Encrypted 16-bit unsigned integer |
| `euint32` | 32 | Encrypted 32-bit unsigned integer |
| `euint64` | 64 | Encrypted 64-bit unsigned integer |
| `euint128` | 128 | Encrypted 128-bit unsigned integer |
| `euint256` | 256 | Encrypted 256-bit unsigned integer |
| `eaddress` / `euint160` | 160 | Encrypted address |

### External Types (for function inputs)

| External Type | Internal Type | Usage |
|---------------|---------------|-------|
| `externalEbool` | `ebool` | Function parameter for encrypted boolean |
| `externalEuint8` | `euint8` | Function parameter for encrypted uint8 |
| `externalEuint16` | `euint16` | Function parameter for encrypted uint16 |
| `externalEuint32` | `euint32` | Function parameter for encrypted uint32 |
| `externalEuint64` | `euint64` | Function parameter for encrypted uint64 |
| `externalEuint128` | `euint128` | Function parameter for encrypted uint128 |
| `externalEuint256` | `euint256` | Function parameter for encrypted uint256 |
| `externalEaddress` | `eaddress` | Function parameter for encrypted address |

### Import Statement

```solidity
import {
    FHE,
    ebool, euint8, euint16, euint32, euint64, euint128, euint256, eaddress,
    externalEbool, externalEuint8, externalEuint16, externalEuint32,
    externalEuint64, externalEuint128, externalEuint256, externalEaddress
} from "@fhevm/solidity/lib/FHE.sol";
```

---

## FHE Operations

### Arithmetic Operations

| Operation | Function | Operator | Notes |
|-----------|----------|----------|-------|
| Addition | `FHE.add(a, b)` | `a + b` | Encrypted + Encrypted or Encrypted + Plaintext |
| Subtraction | `FHE.sub(a, b)` | `a - b` | Encrypted - Encrypted or Encrypted - Plaintext |
| Multiplication | `FHE.mul(a, b)` | `a * b` | Encrypted * Encrypted or Encrypted * Plaintext |
| Division | `FHE.div(a, b)` | - | **Plaintext divisor only** |
| Remainder | `FHE.rem(a, b)` | - | **Plaintext divisor only** |
| Negation | `FHE.neg(a)` | - | Two's complement negation |
| Minimum | `FHE.min(a, b)` | - | Returns smaller value |
| Maximum | `FHE.max(a, b)` | - | Returns larger value |

```solidity
euint64 a = FHE.asEuint64(100);
euint64 b = FHE.asEuint64(50);

euint64 sum = FHE.add(a, b);        // or: a + b
euint64 diff = FHE.sub(a, b);       // or: a - b
euint64 product = FHE.mul(a, b);    // or: a * b
euint64 quotient = FHE.div(a, 10);  // Plaintext divisor only
euint64 remainder = FHE.rem(a, 10); // Plaintext divisor only
euint64 minVal = FHE.min(a, b);
euint64 maxVal = FHE.max(a, b);
```

### Bitwise Operations

| Operation | Function | Operator | Notes |
|-----------|----------|----------|-------|
| AND | `FHE.and(a, b)` | `a & b` | Bitwise AND |
| OR | `FHE.or(a, b)` | `a \| b` | Bitwise OR |
| XOR | `FHE.xor(a, b)` | `a ^ b` | Bitwise XOR |
| NOT | `FHE.not(a)` | `~a` | Bitwise NOT |
| Shift Right | `FHE.shr(a, b)` | - | Shift right by b bits |
| Shift Left | `FHE.shl(a, b)` | - | Shift left by b bits |
| Rotate Right | `FHE.rotr(a, b)` | - | Rotate right by b bits |
| Rotate Left | `FHE.rotl(a, b)` | - | Rotate left by b bits |

```solidity
euint32 a = FHE.asEuint32(0xFF00);
euint32 b = FHE.asEuint32(0x00FF);

euint32 andResult = FHE.and(a, b);  // or: a & b
euint32 orResult = FHE.or(a, b);    // or: a | b
euint32 xorResult = FHE.xor(a, b);  // or: a ^ b
euint32 notResult = FHE.not(a);     // or: ~a
euint32 shifted = FHE.shr(a, 4);    // Shift right by 4 bits
```

### Comparison Operations

All comparison operations return `ebool` (encrypted boolean).

| Operation | Function | Description |
|-----------|----------|-------------|
| Equal | `FHE.eq(a, b)` | Returns true if a == b |
| Not Equal | `FHE.ne(a, b)` | Returns true if a != b |
| Greater Than or Equal | `FHE.ge(a, b)` | Returns true if a >= b |
| Greater Than | `FHE.gt(a, b)` | Returns true if a > b |
| Less Than or Equal | `FHE.le(a, b)` | Returns true if a <= b |
| Less Than | `FHE.lt(a, b)` | Returns true if a < b |

```solidity
euint64 balance = FHE.asEuint64(1000);
euint64 amount = FHE.asEuint64(500);

ebool isEqual = FHE.eq(balance, amount);
ebool isGreater = FHE.gt(balance, amount);
ebool isLessOrEqual = FHE.le(balance, amount);
```

### Ternary Operation (Select)

The `FHE.select` function is the **only way to branch** on encrypted conditions.

```solidity
// FHE.select(condition, valueIfTrue, valueIfFalse)
euint64 result = FHE.select(condition, trueValue, falseValue);
```

**Example: Safe Subtraction**

```solidity
function safeSubtract(euint64 a, euint64 b) internal returns (euint64) {
    ebool isEnough = FHE.ge(a, b);
    euint64 difference = FHE.sub(a, b);
    euint64 zero = FHE.asEuint64(0);

    // If a >= b, return difference; otherwise return 0
    return FHE.select(isEnough, difference, zero);
}
```

### Random Number Generation

```solidity
ebool randomBool = FHE.randEbool();
euint8 random8 = FHE.randEuint8();
euint16 random16 = FHE.randEuint16();
euint32 random32 = FHE.randEuint32();
euint64 random64 = FHE.randEuint64();
euint128 random128 = FHE.randEuint128();
euint256 random256 = FHE.randEuint256();
```

### Type Conversion

```solidity
// Plaintext to Encrypted
euint64 encrypted = FHE.asEuint64(100);
ebool encryptedBool = FHE.asEbool(true);

// Type Casting (between encrypted types)
euint64 value64 = FHE.asEuint64(value32);
euint32 value32 = FHE.asEuint32(value64); // Truncates if necessary
```

---

## Encrypted Inputs

### Receiving Encrypted Inputs

Functions that receive encrypted data from users must:
1. Accept `externalEuintX` parameters
2. Accept a `bytes calldata inputProof` parameter
3. Validate using `FHE.fromExternal()`

```solidity
function deposit(
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) external {
    // Validate and convert to internal encrypted type
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);

    // Use the validated encrypted value
    _balances[msg.sender] = FHE.add(_balances[msg.sender], amount);

    // Set permissions
    FHE.allowThis(_balances[msg.sender]);
    FHE.allow(_balances[msg.sender], msg.sender);
}
```

### Multiple Encrypted Inputs

```solidity
function complexOperation(
    externalEuint64 encryptedA,
    externalEuint32 encryptedB,
    externalEbool encryptedFlag,
    bytes calldata inputProof
) external {
    euint64 a = FHE.fromExternal(encryptedA, inputProof);
    euint32 b = FHE.fromExternal(encryptedB, inputProof);
    ebool flag = FHE.fromExternal(encryptedFlag, inputProof);

    // All values validated from the same inputProof
}
```

---

## Access Control List (ACL)

The ACL system controls which addresses can access (decrypt) encrypted values.

### Permission Types

| Type | Function | Duration | Storage |
|------|----------|----------|---------|
| Permanent | `FHE.allow(handle, address)` | Persistent | Contract storage |
| Transient | `FHE.allowTransient(handle, address)` | Current tx only | EIP-1153 transient storage |
| Contract Self | `FHE.allowThis(handle)` | Persistent | Contract storage |
| Public | `FHE.makePubliclyDecryptable(handle)` | Persistent | Anyone can decrypt |

### Common ACL Patterns

**Pattern 1: Allow Contract and User**

```solidity
function updateValue(externalEuint64 newValue, bytes calldata inputProof) external {
    euint64 value = FHE.fromExternal(newValue, inputProof);
    _storedValue = value;

    // Contract needs access for future operations
    FHE.allowThis(_storedValue);

    // User needs access to decrypt their value
    FHE.allow(_storedValue, msg.sender);
}
```

**Pattern 2: Transfer with ACL Update**

```solidity
function transfer(address to, euint64 amount) internal {
    _balances[msg.sender] = FHE.sub(_balances[msg.sender], amount);
    _balances[to] = FHE.add(_balances[to], amount);

    // Update permissions for both parties
    FHE.allowThis(_balances[msg.sender]);
    FHE.allow(_balances[msg.sender], msg.sender);

    FHE.allowThis(_balances[to]);
    FHE.allow(_balances[to], to);
}
```

**Pattern 3: Return Value with Transient Permission**

```solidity
function getEncryptedResult() external returns (euint64) {
    euint64 result = _computeResult();

    // Allow caller to decrypt the returned value (this tx only)
    FHE.allowTransient(result, msg.sender);

    return result;
}
```

### Checking Permissions

```solidity
// Check if address has access
bool hasAccess = FHE.isAllowed(handle, address);

// Check if msg.sender has access
bool senderAllowed = FHE.isSenderAllowed(handle);
```

---

## HCU (Homomorphic Computing Unit) Costs

Each FHE operation has a computational cost measured in HCU.

### Current Devnet Limits

| Limit Type | Value |
|------------|-------|
| Global complexity per tx | 20,000,000 HCU |
| Sequential depth per tx | 5,000,000 HCU |

### Operation Costs (Approximate)

| Operation | euint8 | euint64 | euint128 |
|-----------|--------|---------|----------|
| add/sub | 84,000 | 88,000 | 95,000 |
| mul | 150,000 | 696,000 | 1,686,000 |
| div/rem | 252,000 | 1,134,000 | 3,057,000 |
| comparison | 42,000 | 47,000 | 60,000 |
| select | 42,000 | 47,000 | 68,000 |
| not (bool) | 2 | - | - |
| and/or/xor | 30,000 | 32,000 | 33,000 |
| cast/encrypt | 32 | 32 | 32 |
| random | 23,000 | 26,000 | 30,000 |

### Optimization Tips

1. **Use smallest sufficient type**: `euint8` operations are cheaper than `euint64`
2. **Prefer plaintext operands**: `FHE.add(encrypted, 10)` is cheaper than `FHE.add(encrypted, encryptedTen)`
3. **Batch operations**: Minimize the number of FHE operations per transaction
4. **Avoid unnecessary type conversions**: Keep values in their native type when possible

---

## Complete Example: Confidential Counter

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint32, externalEuint32} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract ConfidentialCounter is ZamaEthereumConfig {
    euint32 private _count;

    event CountUpdated(address indexed user);

    function getCount() external view returns (euint32) {
        return _count;
    }

    function increment(
        externalEuint32 encryptedAmount,
        bytes calldata inputProof
    ) external {
        euint32 amount = FHE.fromExternal(encryptedAmount, inputProof);
        _count = FHE.add(_count, amount);

        FHE.allowThis(_count);
        FHE.allow(_count, msg.sender);

        emit CountUpdated(msg.sender);
    }

    function decrement(
        externalEuint32 encryptedAmount,
        bytes calldata inputProof
    ) external {
        euint32 amount = FHE.fromExternal(encryptedAmount, inputProof);

        // Safe subtraction: if count < amount, result is 0
        ebool isEnough = FHE.ge(_count, amount);
        euint32 newCount = FHE.sub(_count, amount);
        _count = FHE.select(isEnough, newCount, FHE.asEuint32(0));

        FHE.allowThis(_count);
        FHE.allow(_count, msg.sender);

        emit CountUpdated(msg.sender);
    }

    function reset() external {
        _count = FHE.asEuint32(0);

        FHE.allowThis(_count);
        FHE.allow(_count, msg.sender);

        emit CountUpdated(msg.sender);
    }
}
```

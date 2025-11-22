# Utility Libraries

OpenZeppelin Confidential Contracts provides utility libraries for common operations with encrypted values.

## FHESafeMath

Safe arithmetic operations for encrypted integers that handle overflow/underflow gracefully.

### Import

```solidity
import {FHESafeMath} from "@openzeppelin/confidential-contracts/utils/FHESafeMath.sol";
```

### Why Use FHESafeMath?

Standard FHE arithmetic operations can overflow/underflow silently. FHESafeMath provides "try" variants that return an encrypted boolean indicating success.

```solidity
// Without FHESafeMath - silent overflow
euint64 a = FHE.asEuint64(type(uint64).max);
euint64 b = FHE.asEuint64(1);
euint64 result = FHE.add(a, b); // Overflows silently!

// With FHESafeMath - explicit handling
(ebool success, euint64 safeResult) = FHESafeMath.tryAdd(a, b);
// success = false (encrypted), safeResult = 0
```

### Functions

#### tryAdd

```solidity
function tryAdd(euint64 a, euint64 b) internal returns (ebool success, euint64 result);
```

Adds two encrypted values. Returns `(false, 0)` on overflow.

```solidity
euint64 balance = FHE.asEuint64(1000);
euint64 deposit = FHE.asEuint64(500);

(ebool success, euint64 newBalance) = FHESafeMath.tryAdd(balance, deposit);

// Use select to handle failure
euint64 finalBalance = FHE.select(success, newBalance, balance);
```

#### trySub

```solidity
function trySub(euint64 a, euint64 b) internal returns (ebool success, euint64 result);
```

Subtracts b from a. Returns `(false, 0)` on underflow.

```solidity
euint64 balance = FHE.asEuint64(100);
euint64 withdrawal = FHE.asEuint64(150);

(ebool success, euint64 newBalance) = FHESafeMath.trySub(balance, withdrawal);

// success = false (encrypted) because 100 - 150 underflows
euint64 finalBalance = FHE.select(success, newBalance, balance);
```

#### tryIncrease

```solidity
function tryIncrease(euint64 oldValue, euint64 delta) internal returns (ebool success, euint64 result);
```

Increases a value by delta. Returns `(false, oldValue)` on overflow.

```solidity
euint64 counter = getCounter();
euint64 increment = FHE.asEuint64(10);

(ebool success, euint64 newCounter) = FHESafeMath.tryIncrease(counter, increment);
```

#### tryDecrease

```solidity
function tryDecrease(euint64 oldValue, euint64 delta) internal returns (ebool success, euint64 result);
```

Decreases a value by delta. Returns `(false, oldValue)` on underflow.

```solidity
euint64 inventory = getInventory();
euint64 sold = FHE.asEuint64(5);

(ebool success, euint64 newInventory) = FHESafeMath.tryDecrease(inventory, sold);
```

### Complete Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint64, externalEuint64, ebool} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";
import {FHESafeMath} from "@openzeppelin/confidential-contracts/utils/FHESafeMath.sol";

contract SafeVault is ZamaEthereumConfig {
    mapping(address => euint64) private _balances;

    event DepositResult(address indexed user, bool success);
    event WithdrawResult(address indexed user, bool success);

    function deposit(
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        euint64 currentBalance = _balances[msg.sender];

        // Safe addition - won't overflow
        (ebool success, euint64 newBalance) = FHESafeMath.tryAdd(
            currentBalance,
            amount
        );

        // Only update if successful
        _balances[msg.sender] = FHE.select(success, newBalance, currentBalance);

        FHE.allowThis(_balances[msg.sender]);
        FHE.allow(_balances[msg.sender], msg.sender);
    }

    function withdraw(
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        euint64 currentBalance = _balances[msg.sender];

        // Safe subtraction - won't underflow
        (ebool success, euint64 newBalance) = FHESafeMath.trySub(
            currentBalance,
            amount
        );

        // Only update if successful (sufficient balance)
        _balances[msg.sender] = FHE.select(success, newBalance, currentBalance);

        FHE.allowThis(_balances[msg.sender]);
        FHE.allow(_balances[msg.sender], msg.sender);
    }

    function getBalance() external view returns (euint64) {
        return _balances[msg.sender];
    }
}
```

---

## HandleAccessManager

Abstract contract for managing access to encrypted handles with support for both persistent and transient permissions.

### Import

```solidity
import {HandleAccessManager} from "@openzeppelin/confidential-contracts/utils/HandleAccessManager.sol";
```

### Overview

HandleAccessManager provides a base implementation for contracts that need to manage ACL permissions for encrypted values. It requires implementing a validation function.

### Implementation Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {HandleAccessManager} from "@openzeppelin/confidential-contracts/utils/HandleAccessManager.sol";
import {FHE, euint64} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract MyAccessManagedContract is HandleAccessManager, ZamaEthereumConfig {
    mapping(address => euint64) private _secrets;

    // Required: validate handle belongs to allowed set
    function _validateHandleAllowance(
        address account,
        euint64 handle
    ) internal view override returns (bool) {
        // Only allow access to user's own secret
        return FHE.eq(_secrets[account], handle);
    }

    function setSecret(euint64 secret) external {
        _secrets[msg.sender] = secret;

        // Use HandleAccessManager to grant access
        _grantAccess(msg.sender, secret);
    }

    function getSecret() external view returns (euint64) {
        return _secrets[msg.sender];
    }
}
```

### Key Functions

```solidity
// Grant persistent access to a handle
function _grantAccess(address account, euint64 handle) internal;

// Grant transient access (current transaction only)
function _grantTransientAccess(address account, euint64 handle) internal;

// Revoke access (if supported)
function _revokeAccess(address account, euint64 handle) internal;

// Must implement: validate if handle can be accessed
function _validateHandleAllowance(
    address account,
    euint64 handle
) internal view virtual returns (bool);
```

---

## CheckpointsConfidential

Checkpoint tracking for encrypted values, useful for historical queries (voting power, balances at block).

### Import

```solidity
import {CheckpointsConfidential} from
    "@openzeppelin/confidential-contracts/utils/structs/CheckpointsConfidential.sol";
```

### Structures

```solidity
// 32-bit encrypted value checkpoints
struct TraceEuint32 {
    Checkpoint32[] _checkpoints;
}

struct Checkpoint32 {
    uint48 _key;      // Block number or timestamp
    euint32 _value;   // Encrypted value at that point
}

// 64-bit encrypted value checkpoints
struct TraceEuint64 {
    Checkpoint64[] _checkpoints;
}

struct Checkpoint64 {
    uint48 _key;
    euint64 _value;
}
```

### Functions

#### push

Add a new checkpoint.

```solidity
using CheckpointsConfidential for CheckpointsConfidential.TraceEuint64;

CheckpointsConfidential.TraceEuint64 private _balanceHistory;

function updateBalance(euint64 newBalance) internal {
    _balanceHistory.push(uint48(block.number), newBalance);
}
```

#### latest

Get the most recent value.

```solidity
function getCurrentBalance() external view returns (euint64) {
    return _balanceHistory.latest();
}
```

#### lookup

Get value at specific key (block number).

```solidity
function getBalanceAt(uint48 blockNumber) external view returns (euint64) {
    return _balanceHistory.lookup(blockNumber);
}
```

#### upperLookup / lowerLookup

Binary search for checkpoint.

```solidity
// Find checkpoint >= key
euint64 value = _balanceHistory.upperLookup(blockNumber);

// Find checkpoint <= key
euint64 value = _balanceHistory.lowerLookup(blockNumber);
```

#### length

Get number of checkpoints.

```solidity
uint256 count = _balanceHistory.length();
```

### Complete Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint64, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";
import {CheckpointsConfidential} from
    "@openzeppelin/confidential-contracts/utils/structs/CheckpointsConfidential.sol";

contract HistoricalBalances is ZamaEthereumConfig {
    using CheckpointsConfidential for CheckpointsConfidential.TraceEuint64;

    mapping(address => CheckpointsConfidential.TraceEuint64) private _balanceHistory;
    mapping(address => euint64) private _currentBalances;

    function deposit(
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        euint64 currentBalance = _currentBalances[msg.sender];
        euint64 newBalance = FHE.add(currentBalance, amount);

        _currentBalances[msg.sender] = newBalance;

        // Record checkpoint
        _balanceHistory[msg.sender].push(uint48(block.number), newBalance);

        FHE.allowThis(newBalance);
        FHE.allow(newBalance, msg.sender);
    }

    function getCurrentBalance() external view returns (euint64) {
        return _currentBalances[msg.sender];
    }

    function getBalanceAt(uint48 blockNumber) external view returns (euint64) {
        return _balanceHistory[msg.sender].lookup(blockNumber);
    }

    function getCheckpointCount() external view returns (uint256) {
        return _balanceHistory[msg.sender].length();
    }
}
```

---

## VotesConfidential

Confidential voting power tracking with delegation support. Used by `ERC7984Votes`.

### Import

```solidity
import {VotesConfidential} from
    "@openzeppelin/confidential-contracts/governance/utils/VotesConfidential.sol";
```

### Overview

VotesConfidential tracks encrypted voting power with:
- Historical checkpoints for past vote queries
- Delegation (vote on behalf of)
- EIP-712 signature-based delegation

### Key Functions

```solidity
// Get current voting power (encrypted)
function getVotes(address account) public view returns (euint64);

// Get voting power at past block
function getPastVotes(address account, uint256 blockNumber) public view returns (euint64);

// Get total supply at past block
function getPastTotalSupply(uint256 blockNumber) public view returns (euint64);

// Get current delegate
function delegates(address account) public view returns (address);

// Delegate votes to another address
function delegate(address delegatee) public;

// Delegate by EIP-712 signature
function delegateBySig(
    address delegatee,
    uint256 nonce,
    uint256 expiry,
    uint8 v,
    bytes32 r,
    bytes32 s
) public;
```

### Delegation Pattern

```
┌─────────────┐    delegate(B)    ┌─────────────┐
│   User A    │ ────────────────▶ │   User B    │
│  100 votes  │                   │  (delegate) │
└─────────────┘                   └─────────────┘

A's voting power: 0 (delegated away)
B's voting power: 100 (from A) + B's own tokens
```

### Implementation Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {VotesConfidential} from
    "@openzeppelin/confidential-contracts/governance/utils/VotesConfidential.sol";
import {FHE, euint64} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract GovernanceVoting is VotesConfidential, ZamaEthereumConfig {

    // Voting power comes from token balance
    function _getVotingUnits(address account) internal view override returns (euint64) {
        return _balances[account];
    }

    // Transfer voting power on token transfer
    function _update(
        address from,
        address to,
        euint64 amount
    ) internal override returns (euint64) {
        euint64 transferred = super._update(from, to, amount);

        // Move voting power
        _transferVotingUnits(from, to, amount);

        return transferred;
    }
}
```

---

## Utility Pattern Summary

| Utility | Purpose | Use Case |
|---------|---------|----------|
| `FHESafeMath` | Safe arithmetic | Prevent overflow/underflow in transfers |
| `HandleAccessManager` | ACL management | Custom access control patterns |
| `CheckpointsConfidential` | Historical tracking | Voting power at block, balance history |
| `VotesConfidential` | Governance voting | DAO governance with encrypted votes |

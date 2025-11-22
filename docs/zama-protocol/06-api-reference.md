# API Reference & Cheat Sheet

Quick reference for Zama Protocol FHEVM and OpenZeppelin Confidential Contracts.

---

## Import Cheat Sheet

```solidity
// FHEVM Core
import {FHE, euint64, externalEuint64, ebool} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

// OpenZeppelin ERC7984
import {ERC7984} from "@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol";
import {IERC7984} from "@openzeppelin/confidential-contracts/interfaces/IERC7984.sol";

// Extensions
import {ERC7984ERC20Wrapper} from "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984ERC20Wrapper.sol";
import {ERC7984Freezable} from "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Freezable.sol";
import {ERC7984Restricted} from "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Restricted.sol";
import {ERC7984Rwa} from "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Rwa.sol";
import {ERC7984Votes} from "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Votes.sol";
import {ERC7984ObserverAccess} from "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984ObserverAccess.sol";

// Utilities
import {FHESafeMath} from "@openzeppelin/confidential-contracts/utils/FHESafeMath.sol";
import {CheckpointsConfidential} from "@openzeppelin/confidential-contracts/utils/structs/CheckpointsConfidential.sol";
import {VotesConfidential} from "@openzeppelin/confidential-contracts/governance/utils/VotesConfidential.sol";
```

---

## Encrypted Types Reference

| Type | External Type | Bits | Max Value |
|------|---------------|------|-----------|
| `ebool` | `externalEbool` | 2 | true/false |
| `euint8` | `externalEuint8` | 8 | 255 |
| `euint16` | `externalEuint16` | 16 | 65,535 |
| `euint32` | `externalEuint32` | 32 | 4,294,967,295 |
| `euint64` | `externalEuint64` | 64 | 18,446,744,073,709,551,615 |
| `euint128` | `externalEuint128` | 128 | 2^128 - 1 |
| `euint256` | `externalEuint256` | 256 | 2^256 - 1 |
| `eaddress` | `externalEaddress` | 160 | Address |

---

## FHE Library Functions

### Conversion

| Function | Description | Example |
|----------|-------------|---------|
| `FHE.asEuint8(x)` | Convert to euint8 | `FHE.asEuint8(100)` |
| `FHE.asEuint16(x)` | Convert to euint16 | `FHE.asEuint16(1000)` |
| `FHE.asEuint32(x)` | Convert to euint32 | `FHE.asEuint32(100000)` |
| `FHE.asEuint64(x)` | Convert to euint64 | `FHE.asEuint64(1e18)` |
| `FHE.asEbool(x)` | Convert to ebool | `FHE.asEbool(true)` |
| `FHE.fromExternal(ext, proof)` | Validate external input | `FHE.fromExternal(encAmount, proof)` |

### Arithmetic

| Function | Operator | Description |
|----------|----------|-------------|
| `FHE.add(a, b)` | `a + b` | Addition |
| `FHE.sub(a, b)` | `a - b` | Subtraction |
| `FHE.mul(a, b)` | `a * b` | Multiplication |
| `FHE.div(a, b)` | - | Division (plaintext b only) |
| `FHE.rem(a, b)` | - | Remainder (plaintext b only) |
| `FHE.neg(a)` | - | Negation |
| `FHE.min(a, b)` | - | Minimum |
| `FHE.max(a, b)` | - | Maximum |

### Bitwise

| Function | Operator | Description |
|----------|----------|-------------|
| `FHE.and(a, b)` | `a & b` | Bitwise AND |
| `FHE.or(a, b)` | `a \| b` | Bitwise OR |
| `FHE.xor(a, b)` | `a ^ b` | Bitwise XOR |
| `FHE.not(a)` | `~a` | Bitwise NOT |
| `FHE.shr(a, b)` | - | Shift right |
| `FHE.shl(a, b)` | - | Shift left |
| `FHE.rotr(a, b)` | - | Rotate right |
| `FHE.rotl(a, b)` | - | Rotate left |

### Comparison (returns `ebool`)

| Function | Description |
|----------|-------------|
| `FHE.eq(a, b)` | Equal |
| `FHE.ne(a, b)` | Not equal |
| `FHE.gt(a, b)` | Greater than |
| `FHE.ge(a, b)` | Greater than or equal |
| `FHE.lt(a, b)` | Less than |
| `FHE.le(a, b)` | Less than or equal |

### Control Flow

| Function | Description | Example |
|----------|-------------|---------|
| `FHE.select(cond, a, b)` | Ternary selection | `FHE.select(isValid, value, zero)` |

### Random

| Function | Description |
|----------|-------------|
| `FHE.randEbool()` | Random encrypted boolean |
| `FHE.randEuint8()` | Random encrypted uint8 |
| `FHE.randEuint16()` | Random encrypted uint16 |
| `FHE.randEuint32()` | Random encrypted uint32 |
| `FHE.randEuint64()` | Random encrypted uint64 |
| `FHE.randEuint128()` | Random encrypted uint128 |
| `FHE.randEuint256()` | Random encrypted uint256 |

### ACL (Access Control)

| Function | Description |
|----------|-------------|
| `FHE.allow(handle, address)` | Grant permanent access |
| `FHE.allowTransient(handle, address)` | Grant access for current tx only |
| `FHE.allowThis(handle)` | Grant access to current contract |
| `FHE.makePubliclyDecryptable(handle)` | Allow anyone to decrypt |
| `FHE.isAllowed(handle, address)` | Check if address has access |
| `FHE.isSenderAllowed(handle)` | Check if msg.sender has access |

---

## IERC7984 Interface

### View Functions

```solidity
function name() external view returns (string memory);
function symbol() external view returns (string memory);
function decimals() external view returns (uint8);
function confidentialTotalSupply() external view returns (euint64);
function confidentialBalanceOf(address account) external view returns (euint64);
function isOperator(address holder, address spender) external view returns (bool);
function operatorExpiration(address holder, address operator) external view returns (uint48);
```

### State-Changing Functions

```solidity
function confidentialTransfer(
    address to,
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) external returns (euint64 transferred);

function confidentialTransferFrom(
    address from,
    address to,
    euint64 amount
) external returns (euint64 transferred);

function confidentialTransferAndCall(
    address to,
    externalEuint64 encryptedAmount,
    bytes calldata inputProof,
    bytes calldata data
) external returns (euint64 transferred);

function confidentialTransferFromAndCall(
    address from,
    address to,
    euint64 amount,
    bytes calldata data
) external returns (euint64 transferred);

function setOperator(address operator, uint48 until) external;

function requestDiscloseEncryptedAmount(euint64 amount) external;

function discloseEncryptedAmount(euint64 encryptedAmount, uint64 amount) external;
```

---

## FHESafeMath Functions

```solidity
function tryAdd(euint64 a, euint64 b) returns (ebool success, euint64 result);
function trySub(euint64 a, euint64 b) returns (ebool success, euint64 result);
function tryIncrease(euint64 oldValue, euint64 delta) returns (ebool success, euint64 result);
function tryDecrease(euint64 oldValue, euint64 delta) returns (ebool success, euint64 result);
```

---

## Common Patterns

### Pattern 1: Receive and Validate Encrypted Input

```solidity
function myFunction(
    externalEuint64 encryptedValue,
    bytes calldata inputProof
) external {
    euint64 value = FHE.fromExternal(encryptedValue, inputProof);
    // Use value...
}
```

### Pattern 2: Safe Transfer with Balance Check

```solidity
function safeTransfer(address to, euint64 amount) internal {
    euint64 senderBalance = _balances[msg.sender];

    // Check sufficient balance
    ebool hasEnough = FHE.ge(senderBalance, amount);

    // Calculate new balances
    euint64 newSenderBalance = FHE.sub(senderBalance, amount);
    euint64 newReceiverBalance = FHE.add(_balances[to], amount);

    // Only update if has enough
    _balances[msg.sender] = FHE.select(hasEnough, newSenderBalance, senderBalance);
    _balances[to] = FHE.select(hasEnough, newReceiverBalance, _balances[to]);

    // Update ACL
    FHE.allowThis(_balances[msg.sender]);
    FHE.allow(_balances[msg.sender], msg.sender);
    FHE.allowThis(_balances[to]);
    FHE.allow(_balances[to], to);
}
```

### Pattern 3: Store and Return with ACL

```solidity
function setAndGet(externalEuint64 encryptedValue, bytes calldata inputProof)
    external
    returns (euint64)
{
    euint64 value = FHE.fromExternal(encryptedValue, inputProof);

    _storedValue = value;

    // Persistent access for contract
    FHE.allowThis(_storedValue);

    // Persistent access for user
    FHE.allow(_storedValue, msg.sender);

    // Transient access for return value
    FHE.allowTransient(_storedValue, msg.sender);

    return _storedValue;
}
```

### Pattern 4: Comparison with Branching

```solidity
function conditionalUpdate(euint64 newValue) internal {
    euint64 currentValue = _value;

    // Encrypted comparison
    ebool isGreater = FHE.gt(newValue, currentValue);

    // Select based on condition
    _value = FHE.select(isGreater, newValue, currentValue);

    FHE.allowThis(_value);
}
```

---

## Relayer SDK Quick Reference

### Initialization

```typescript
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk";

// Simple setup with pre-configured Sepolia
const instance = await createInstance(SepoliaConfig);

// Or with custom configuration
const instance = await createInstance({
  aclContractAddress: "0x687820221192C5B662b25367F70076A37bc79b6c",
  kmsContractAddress: "0x1364cBBf2cDF5032C47d8226a6f6FBD2AFCDacAC",
  inputVerifierContractAddress: "0xbc91f3daD1A5F19F8390c400196e58073B6a0BC4",
  verifyingContractAddressDecryption: "0xb6E160B1ff80D67Bfe90A85eE06Ce0A2613607D1",
  verifyingContractAddressInputVerification: "0x7048C39f048125eDa9d678AEbaDfB22F7900a29F",
  chainId: 11155111,
  gatewayChainId: 55815,
  network: "https://eth-sepolia.public.blastapi.io",
  relayerUrl: "https://relayer.testnet.zama.cloud",
});
```

### Create Encrypted Input

```typescript
const input = instance.createEncryptedInput(contractAddress, userAddress);
input.add64(BigInt(amount));
const encrypted = await input.encrypt();

// Use in contract call
await contract.myFunction(encrypted.handles[0], encrypted.inputProof);
```

### User Decryption

```typescript
const keypair = instance.generateKeyPair();
const eip712Message = instance.createEIP712Message(handle, keypair.publicKey);
const signature = await signer.signTypedData(
  eip712Message.domain,
  eip712Message.types,
  eip712Message.message
);
const decrypted = await instance.userDecrypt(keypair, signature, handle);
```

### Public Decryption

```typescript
const value = await instance.publicDecrypt(handle);
```

---

## HCU Costs Quick Reference

| Operation | euint8 | euint64 |
|-----------|--------|---------|
| add/sub | ~84k | ~88k |
| mul | ~150k | ~696k |
| div/rem | ~252k | ~1.1M |
| compare | ~42k | ~47k |
| select | ~42k | ~47k |
| and/or/xor | ~30k | ~32k |
| random | ~23k | ~26k |
| cast | ~32 | ~32 |

**Limits:**
- Global: 20,000,000 HCU per tx
- Sequential: 5,000,000 HCU per tx

---

## Contract Template

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC7984} from "@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol";
import {FHE, euint64, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract MyConfidentialToken is ERC7984, ZamaEthereumConfig {

    constructor() ERC7984("My Token", "MTK") {}

    function mint(
        address to,
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        _mint(to, amount);
    }

    function burn(
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        _burn(msg.sender, amount);
    }
}
```

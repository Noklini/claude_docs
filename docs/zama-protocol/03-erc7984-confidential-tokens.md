# ERC7984 Confidential Token Standard

## Overview

ERC7984 is OpenZeppelin's implementation of a confidential fungible token standard built on Zama's fhEVM. It provides encrypted balances, confidential transfers, and a flexible operator/disclosure system.

> **Warning**: OpenZeppelin Confidential Contracts is experimental and unaudited. Use at your own risk.

## Installation

```bash
npm install @openzeppelin/confidential-contracts
```

## Import Structure

```solidity
// Core interfaces
import {IERC7984} from "@openzeppelin/confidential-contracts/interfaces/IERC7984.sol";
import {IERC7984Receiver} from "@openzeppelin/confidential-contracts/interfaces/IERC7984Receiver.sol";

// Base implementation
import {ERC7984} from "@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol";

// FHEVM imports
import {FHE, euint64, externalEuint64, ebool} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";
```

---

## Core Interface (IERC7984)

### State Query Functions

```solidity
// Get encrypted balance of an account
function confidentialBalanceOf(address account) external view returns (euint64);

// Get encrypted total supply
function confidentialTotalSupply() external view returns (euint64);

// Get token name
function name() external view returns (string memory);

// Get token symbol
function symbol() external view returns (string memory);

// Get decimals (default: 6 for confidential tokens)
function decimals() external view returns (uint8);
```

### Transfer Functions

```solidity
// Transfer with encrypted amount input
function confidentialTransfer(
    address to,
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) external returns (euint64 transferred);

// Transfer using internal encrypted handle
function confidentialTransferFrom(
    address from,
    address to,
    euint64 amount
) external returns (euint64 transferred);

// Transfer with receiver callback
function confidentialTransferAndCall(
    address to,
    externalEuint64 encryptedAmount,
    bytes calldata inputProof,
    bytes calldata data
) external returns (euint64 transferred);

// Transfer using internal handle with receiver callback
function confidentialTransferFromAndCall(
    address from,
    address to,
    euint64 amount,
    bytes calldata data
) external returns (euint64 transferred);
```

### Operator System

Operators are addresses authorized to transfer tokens on behalf of holders, with time-bounded permissions.

```solidity
// Set operator with expiration timestamp
function setOperator(address operator, uint48 until) external;

// Check if address is valid operator
function isOperator(address holder, address spender) external view returns (bool);

// Get operator expiration timestamp
function operatorExpiration(address holder, address operator) external view returns (uint48);
```

### Disclosure System

The disclosure system allows selective revelation of encrypted amounts.

```solidity
// Request to disclose an encrypted amount
function requestDiscloseEncryptedAmount(euint64 amount) external;

// Confirm disclosure with the revealed value
function discloseEncryptedAmount(euint64 encryptedAmount, uint64 amount) external;
```

---

## Base Implementation

### Creating a Basic Confidential Token

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC7984} from "@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol";
import {FHE, euint64, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract MyConfidentialToken is ERC7984, ZamaEthereumConfig {

    constructor(
        string memory name_,
        string memory symbol_
    ) ERC7984(name_, symbol_) {}

    // Public mint function (for demo - restrict in production!)
    function mint(
        address to,
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        _mint(to, amount);
    }

    // Public burn function
    function burn(
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        _burn(msg.sender, amount);
    }
}
```

### Internal Functions

```solidity
// Mint tokens to address
function _mint(address to, euint64 amount) internal virtual;

// Burn tokens from address
function _burn(address from, euint64 amount) internal virtual;

// Internal transfer logic
function _transfer(
    address from,
    address to,
    euint64 amount
) internal virtual returns (euint64 transferred);

// Update balances (called by _transfer, _mint, _burn)
function _update(
    address from,
    address to,
    euint64 amount
) internal virtual returns (euint64 transferred);
```

---

## Operator System Usage

### Setting Operators

```solidity
// Grant operator permission for 30 days
uint48 expiration = uint48(block.timestamp + 30 days);
token.setOperator(operatorAddress, expiration);

// Revoke operator (set expiration to 0 or past timestamp)
token.setOperator(operatorAddress, 0);
```

### Operator Transfer Pattern

```solidity
contract TokenVault {
    IERC7984 public token;

    function depositFrom(
        address from,
        euint64 amount
    ) external {
        // Requires: token.isOperator(from, address(this)) == true
        token.confidentialTransferFrom(from, address(this), amount);
    }
}
```

### Checking Operator Status

```solidity
// Check if operator is currently valid
bool isValid = token.isOperator(holder, spender);

// Get expiration timestamp
uint48 expires = token.operatorExpiration(holder, spender);
bool willExpireSoon = expires < block.timestamp + 7 days;
```

---

## Receiver Interface

Contracts receiving tokens via `confidentialTransferAndCall` must implement `IERC7984Receiver`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC7984Receiver} from "@openzeppelin/confidential-contracts/interfaces/IERC7984Receiver.sol";
import {FHE, euint64} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract TokenReceiver is IERC7984Receiver, ZamaEthereumConfig {

    event TokensReceived(
        address indexed operator,
        address indexed from,
        address indexed to,
        bytes data
    );

    function onERC7984Received(
        address operator,
        address from,
        address to,
        euint64 amount,
        bytes calldata data
    ) external override returns (bytes4) {
        // Handle received tokens
        // amount is the encrypted transfer amount

        emit TokensReceived(operator, from, to, data);

        // Return selector to confirm receipt
        return IERC7984Receiver.onERC7984Received.selector;
    }
}
```

---

## Disclosure Pattern

The disclosure system allows users to prove the plaintext value of an encrypted amount.

### Step 1: Request Disclosure

```solidity
// User calls from frontend after getting the plaintext value
token.requestDiscloseEncryptedAmount(encryptedAmount);
```

### Step 2: Confirm Disclosure

```solidity
// After decryption, confirm the value
token.discloseEncryptedAmount(encryptedAmount, plaintextValue);
```

### Custom Disclosure Hook

```solidity
contract MyToken is ERC7984, ZamaEthereumConfig {

    mapping(address => uint64) public disclosedBalances;

    function _afterDisclose(
        address account,
        euint64 encryptedAmount,
        uint64 amount
    ) internal virtual override {
        // Store disclosed balance for public viewing
        disclosedBalances[account] = amount;
    }
}
```

---

## Complete Token Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC7984} from "@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol";
import {FHE, euint64, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract ConfidentialUSD is ERC7984, ZamaEthereumConfig {
    address public owner;

    error OnlyOwner();
    error ZeroAddress();

    modifier onlyOwner() {
        if (msg.sender != owner) revert OnlyOwner();
        _;
    }

    constructor() ERC7984("Confidential USD", "cUSD") {
        owner = msg.sender;
    }

    function mint(
        address to,
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external onlyOwner {
        if (to == address(0)) revert ZeroAddress();
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

    // Override decimals if needed (default is 6)
    function decimals() public pure override returns (uint8) {
        return 6;
    }
}
```

---

## ACL Patterns in ERC7984

The ERC7984 implementation automatically manages ACL permissions:

### On Transfer

```solidity
function _update(address from, address to, euint64 amount) internal override returns (euint64) {
    // ... transfer logic ...

    // Automatically grants permissions:
    // - Contract can access new balances (allowThis)
    // - from can access their new balance (allow)
    // - to can access their new balance (allow)
}
```

### On Mint

```solidity
function _mint(address to, euint64 amount) internal {
    // After minting:
    // - Contract can access recipient's balance
    // - Recipient can access their balance
}
```

### Manual Permission Granting

```solidity
// If you need to grant additional access
function grantBalanceAccess(address account, address viewer) external {
    euint64 balance = _balances[account];
    FHE.allow(balance, viewer);
}
```

---

## Events

```solidity
// Emitted on all transfers (including mint/burn)
event ConfidentialTransfer(
    address indexed from,
    address indexed to
);

// Emitted when operator is set
event OperatorSet(
    address indexed holder,
    address indexed operator,
    uint48 until
);

// Emitted on disclosure request
event DiscloseRequest(
    address indexed account,
    euint64 indexed encryptedAmount
);

// Emitted on disclosure confirmation
event Disclosed(
    address indexed account,
    euint64 indexed encryptedAmount,
    uint64 amount
);
```

---

## Security Considerations

1. **Balance Privacy**: Balances are encrypted, but transfer events are public (sender/receiver visible)
2. **Operator Trust**: Operators have full transfer authority until expiration
3. **Disclosure Irreversibility**: Once disclosed, plaintext values are publicly visible
4. **Overflow Protection**: Use `FHESafeMath` for arithmetic operations
5. **ACL Management**: Ensure proper permissions are granted for all stored values

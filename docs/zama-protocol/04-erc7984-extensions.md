# ERC7984 Extensions

OpenZeppelin Confidential Contracts provides several extensions to the base ERC7984 token standard for common use cases.

## Extension Overview

| Extension | Purpose |
|-----------|---------|
| `ERC7984ERC20Wrapper` | Wrap standard ERC20 tokens into confidential tokens |
| `ERC7984Freezable` | Freeze/unfreeze account balances |
| `ERC7984Restricted` | Blocklist/allowlist transfer restrictions |
| `ERC7984Rwa` | Real World Assets with agent controls |
| `ERC7984Votes` | Governance voting power integration |
| `ERC7984ObserverAccess` | Designated observers for auditing |
| `ERC7984Omnibus` | Omnibus account management |

---

## ERC7984ERC20Wrapper

Wraps standard ERC20 tokens into confidential ERC7984 tokens and vice versa.

### Import

```solidity
import {ERC7984ERC20Wrapper} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984ERC20Wrapper.sol";
```

### Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC7984ERC20Wrapper} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984ERC20Wrapper.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract WrappedConfidentialUSDC is ERC7984ERC20Wrapper, ZamaEthereumConfig {

    constructor(IERC20 underlyingToken)
        ERC7984ERC20Wrapper(underlyingToken)
    {}
}
```

### Key Functions

```solidity
// Get the underlying ERC20 token
function underlying() external view returns (IERC20);

// Wrap ERC20 tokens into confidential tokens
// Requires prior approval: underlying.approve(wrapper, amount)
function wrap(uint256 amount) external;

// Request unwrap (initiates decryption)
function requestUnwrap(
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) external;

// Complete unwrap after decryption
// Called by oracle/relayer with decrypted value
function unwrap(euint64 encryptedAmount, uint64 amount) external;

// Decimal adjustment between ERC20 and ERC7984
function decimalOffset() external view returns (uint8);
```

### Wrap/Unwrap Flow

```
┌─────────────┐       wrap()        ┌─────────────────┐
│   ERC20     │ ─────────────────▶ │  ERC7984        │
│  (public)   │                     │ (confidential)  │
└─────────────┘                     └─────────────────┘
       ▲                                    │
       │                                    │
       │       requestUnwrap()              │
       │       unwrap()                     │
       └────────────────────────────────────┘
```

### Usage Example

```solidity
// 1. Approve wrapper to spend ERC20
usdc.approve(address(wrapper), 1000 * 10**6);

// 2. Wrap into confidential tokens
wrapper.wrap(1000 * 10**6);

// 3. Use confidential tokens...

// 4. Request unwrap (from frontend)
const input = instance.createEncryptedInput(wrapperAddress, userAddress);
input.add64(BigInt(500 * 10**6));
const encrypted = await input.encrypt();

await wrapper.requestUnwrap(encrypted.handles[0], encrypted.inputProof);

// 5. Oracle/relayer completes unwrap after decryption
```

---

## ERC7984Freezable

Enables freezing of account balances, useful for compliance and security.

### Import

```solidity
import {ERC7984Freezable} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Freezable.sol";
```

### Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC7984Freezable} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Freezable.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";
import {FHE, euint64, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";

contract FreezableToken is ERC7984Freezable, ZamaEthereumConfig {
    address public admin;

    modifier onlyAdmin() {
        require(msg.sender == admin, "Not admin");
        _;
    }

    constructor() ERC7984Freezable("Freezable Token", "FRZ") {
        admin = msg.sender;
    }

    // Freeze specific amount
    function freeze(
        address account,
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external onlyAdmin {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        _freeze(account, amount);
    }

    // Unfreeze specific amount
    function unfreeze(
        address account,
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external onlyAdmin {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        _unfreeze(account, amount);
    }
}
```

### Key Functions

```solidity
// Get frozen balance (encrypted)
function frozenBalanceOf(address account) external view returns (euint64);

// Get available (unfrozen) balance (encrypted)
function availableBalanceOf(address account) external view returns (euint64);

// Internal: freeze amount
function _freeze(address account, euint64 amount) internal;

// Internal: unfreeze amount
function _unfreeze(address account, euint64 amount) internal;
```

### Balance Relationship

```
Total Balance = Frozen Balance + Available Balance

Transfers only use Available Balance
```

---

## ERC7984Restricted

Implements blocklist/allowlist functionality for transfer restrictions.

### Import

```solidity
import {ERC7984Restricted} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Restricted.sol";
```

### Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC7984Restricted} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Restricted.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract RestrictedToken is ERC7984Restricted, ZamaEthereumConfig {
    address public complianceOfficer;

    modifier onlyCompliance() {
        require(msg.sender == complianceOfficer, "Not compliance");
        _;
    }

    constructor() ERC7984Restricted("Restricted Token", "RST") {
        complianceOfficer = msg.sender;
    }

    function blockAccount(address account) external onlyCompliance {
        _setRestriction(account, RestrictionState.BLOCKED);
    }

    function allowAccount(address account) external onlyCompliance {
        _setRestriction(account, RestrictionState.ALLOWED);
    }

    function clearRestriction(address account) external onlyCompliance {
        _setRestriction(account, RestrictionState.DEFAULT);
    }
}
```

### Restriction States

```solidity
enum RestrictionState {
    DEFAULT,  // Normal - follows default policy
    BLOCKED,  // Cannot send or receive
    ALLOWED   // Explicitly allowed (useful in allowlist mode)
}
```

### Key Functions

```solidity
// Get account restriction state
function restriction(address account) external view returns (RestrictionState);

// Check if transfer is allowed
function isTransferAllowed(address from, address to) external view returns (bool);

// Internal: set restriction
function _setRestriction(address account, RestrictionState state) internal;
```

### Events

```solidity
event RestrictionSet(address indexed account, RestrictionState state);
```

---

## ERC7984Rwa (Real World Assets)

Designed for tokenized real-world assets with agent-controlled operations.

### Import

```solidity
import {ERC7984Rwa} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Rwa.sol";
```

### Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC7984Rwa} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Rwa.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";
import {FHE, euint64, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";

contract RealEstateToken is ERC7984Rwa, ZamaEthereumConfig {

    constructor(address defaultAdmin, address agent)
        ERC7984Rwa("Real Estate Token", "RET", defaultAdmin, agent)
    {}

    // Agent can mint new tokens
    function issue(
        address to,
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external onlyRole(AGENT_ROLE) {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        _mint(to, amount);
    }

    // Agent can redeem (burn) tokens
    function redeem(
        address from,
        externalEuint64 encryptedAmount,
        bytes calldata inputProof
    ) external onlyRole(AGENT_ROLE) {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        _burn(from, amount);
    }
}
```

### Roles

```solidity
bytes32 public constant AGENT_ROLE = keccak256("AGENT_ROLE");
bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;
```

### Key Functions

```solidity
// Force transfer (agent only, bypasses restrictions)
function forceTransfer(
    address from,
    address to,
    euint64 amount
) external onlyRole(AGENT_ROLE);

// Block account
function blockAccount(address account) external onlyRole(AGENT_ROLE);

// Unblock account
function unblockAccount(address account) external onlyRole(AGENT_ROLE);

// Pause all transfers
function pause() external onlyRole(DEFAULT_ADMIN_ROLE);

// Unpause transfers
function unpause() external onlyRole(DEFAULT_ADMIN_ROLE);
```

---

## ERC7984Votes

Integrates ERC7984 tokens with governance voting systems.

### Import

```solidity
import {ERC7984Votes} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Votes.sol";
```

### Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC7984Votes} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Votes.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract GovernanceToken is ERC7984Votes, ZamaEthereumConfig {

    constructor()
        ERC7984Votes("Governance Token", "GOV")
    {}
}
```

### Key Functions

```solidity
// Get current voting power (encrypted)
function getVotes(address account) external view returns (euint64);

// Get historical voting power at specific block
function getPastVotes(
    address account,
    uint256 blockNumber
) external view returns (euint64);

// Get total supply at specific block
function getPastTotalSupply(uint256 blockNumber) external view returns (euint64);

// Delegate voting power
function delegate(address delegatee) external;

// Delegate by signature (EIP-712)
function delegateBySig(
    address delegatee,
    uint256 nonce,
    uint256 expiry,
    uint8 v,
    bytes32 r,
    bytes32 s
) external;

// Get current delegate
function delegates(address account) external view returns (address);
```

### Voting Power Transfer

Voting power automatically transfers with token transfers:

```
Token Transfer: A → B
Voting Power: A's delegate loses power, B's delegate gains power
```

---

## ERC7984ObserverAccess

Allows designated observers to view encrypted balances and transfers.

### Import

```solidity
import {ERC7984ObserverAccess} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984ObserverAccess.sol";
```

### Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC7984ObserverAccess} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984ObserverAccess.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract AuditableToken is ERC7984ObserverAccess, ZamaEthereumConfig {
    address public admin;

    constructor() ERC7984ObserverAccess("Auditable Token", "AUD") {
        admin = msg.sender;
    }

    function addObserver(address observer) external {
        require(msg.sender == admin, "Not admin");
        _addObserver(observer);
    }

    function removeObserver(address observer) external {
        require(msg.sender == admin, "Not admin");
        _removeObserver(observer);
    }
}
```

### Key Functions

```solidity
// Check if address is an observer
function isObserver(address account) external view returns (bool);

// Internal: add observer
function _addObserver(address observer) internal;

// Internal: remove observer
function _removeObserver(address observer) internal;
```

### Observer Capabilities

Observers automatically receive ACL permissions to decrypt:
- All account balances
- All transfer amounts

---

## Combining Extensions

Extensions can be combined for complex requirements:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC7984} from "@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol";
import {ERC7984Freezable} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Freezable.sol";
import {ERC7984Restricted} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984Restricted.sol";
import {ERC7984ObserverAccess} from
    "@openzeppelin/confidential-contracts/token/ERC7984/extensions/ERC7984ObserverAccess.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";
import {FHE, euint64} from "@fhevm/solidity/lib/FHE.sol";

contract ComplianceToken is
    ERC7984Freezable,
    ERC7984Restricted,
    ERC7984ObserverAccess,
    ZamaEthereumConfig
{
    constructor()
        ERC7984("Compliance Token", "CMP")
    {}

    // Override _update to combine all checks
    function _update(
        address from,
        address to,
        euint64 amount
    ) internal virtual override(ERC7984, ERC7984Freezable, ERC7984Restricted) returns (euint64) {
        // Freezable checks available balance
        // Restricted checks blocklist/allowlist
        return super._update(from, to, amount);
    }
}
```

---

## Extension Inheritance Diagram

```
                    ERC7984 (Base)
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   Freezable        Restricted          Votes
        │                │                │
        └────────┬───────┘                │
                 │                        │
                Rwa                       │
                 │                        │
                 └────────────────────────┘
                           │
                    ObserverAccess
```

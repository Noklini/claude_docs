# Zama Protocol Integration Overview

## Introduction

FHEVM (Fully Homomorphic Encryption Virtual Machine) is Zama's protocol that enables writing confidential smart contracts in Solidity. These contracts operate directly on encrypted data without ever decrypting it onchain, providing end-to-end encryption for blockchain applications.

## Key Guarantees

| Guarantee | Description |
|-----------|-------------|
| **End-to-End Encryption** | Data remains encrypted throughout the entire transaction lifecycle |
| **Composability** | Encrypted values can be used across multiple contracts seamlessly |
| **Non-Disruptive** | Works with existing Solidity patterns and tooling |
| **Quantum-Resistant** | FHE cryptography provides post-quantum security |

## Architecture Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        FHEVM Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   Frontend   │───▶│   Relayer    │───▶│   Gateway    │       │
│  │  (SDK/dApp)  │    │   Service    │    │    Chain     │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│         │                                        │               │
│         │                                        ▼               │
│         │                                ┌──────────────┐        │
│         │                                │     KMS      │        │
│         │                                │ (Key Mgmt)   │        │
│         │                                └──────────────┘        │
│         │                                        │               │
│         ▼                                        ▼               │
│  ┌──────────────────────────────────────────────────────┐       │
│  │                    Host Chain (EVM)                   │       │
│  │  ┌────────────────┐    ┌────────────────────────┐    │       │
│  │  │ Smart Contract │◀──▶│     Coprocessor        │    │       │
│  │  │  (FHE Logic)   │    │  (FHE Computations)    │    │       │
│  │  └────────────────┘    └────────────────────────┘    │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Component Descriptions

| Component | Role |
|-----------|------|
| **FHE Library** | Core cryptographic engine providing homomorphic operations |
| **Host Contracts** | Solidity smart contracts on the blockchain containing FHE logic |
| **Coprocessor** | Off-chain computation component that executes FHE operations |
| **Gateway** | Interface between on-chain contracts and off-chain systems |
| **KMS (Key Management System)** | Securely manages encryption keys for the network |
| **Relayer** | HTTP service that handles all Gateway chain interactions |
| **Oracle** | Verifies and relays computation results back to contracts |

## Development Environment Setup

### Prerequisites

- Node.js >= 18
- npm or pnpm
- Hardhat (recommended) or Foundry (experimental)

### Hardhat Setup (Recommended)

```bash
# Create new project
mkdir my-fhevm-project && cd my-fhevm-project
npm init -y

# Install dependencies
npm install --save-dev hardhat @fhevm/hardhat-plugin @fhevm/solidity

# Initialize Hardhat
npx hardhat init
```

### hardhat.config.ts

```typescript
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "@fhevm/hardhat-plugin";

const config: HardhatUserConfig = {
  solidity: "0.8.24",
  networks: {
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL,
      accounts: [process.env.PRIVATE_KEY!],
    },
  },
};

export default config;
```

### Install OpenZeppelin Confidential Contracts

```bash
npm install @openzeppelin/confidential-contracts
```

## Quick Start Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint32, externalEuint32} from "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract HelloFHEVM is ZamaEthereumConfig {
    euint32 private _secretValue;

    function setSecret(
        externalEuint32 encryptedValue,
        bytes calldata inputProof
    ) external {
        // Validate and convert external encrypted input
        euint32 value = FHE.fromExternal(encryptedValue, inputProof);

        _secretValue = value;

        // Grant access permissions
        FHE.allowThis(_secretValue);
        FHE.allow(_secretValue, msg.sender);
    }

    function getSecret() external view returns (euint32) {
        return _secretValue;
    }
}
```

## Version Information

This documentation covers:

- **FHEVM Protocol**: v0.9
- **OpenZeppelin Confidential Contracts**: Latest (experimental, unaudited)
- **Relayer SDK**: @zama-fhe/relayer-sdk

## Documentation Structure

| Document | Description |
|----------|-------------|
| [01-solidity-workflow.md](./01-solidity-workflow.md) | Core Solidity development with FHE operations |
| [02-relayer-sdk.md](./02-relayer-sdk.md) | Frontend integration with Relayer SDK |
| [03-erc7984-confidential-tokens.md](./03-erc7984-confidential-tokens.md) | OpenZeppelin ERC7984 token standard |
| [04-erc7984-extensions.md](./04-erc7984-extensions.md) | ERC7984 extension contracts |
| [05-utilities.md](./05-utilities.md) | Utility libraries (FHESafeMath, etc.) |
| [06-api-reference.md](./06-api-reference.md) | Quick reference and cheat sheet |

## External Resources

- [Zama Protocol Documentation](https://docs.zama.org/protocol)
- [OpenZeppelin Confidential Contracts](https://github.com/OpenZeppelin/openzeppelin-confidential-contracts)
- [FHEVM Hardhat Template](https://github.com/zama-ai/fhevm-hardhat-template)

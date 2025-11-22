# CLAUDE.md Template for Zama FHEVM Projects

> Copy this file to your project root as `CLAUDE.md` and customize the sections marked with `[CUSTOMIZE]`.

---

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# NEVER USE DEMO OR MOCK DATA

## Repository Overview

[CUSTOMIZE] Brief description of your project.

This is a **[Project Name]** built on Zama Protocol FHEVM (Fully Homomorphic Encryption Virtual Machine). Data is encrypted end-to-end using FHE.

### Project Structure
```
[CUSTOMIZE] Update paths for your project

├── contracts/
│   └── [YourContract].sol      # FHE-encrypted smart contract
├── scripts/
│   └── deploy.ts               # Hardhat deployment script
├── frontend/src/
│   ├── App.tsx                 # Main React app
│   ├── core/
│   │   └── fhevm.ts            # Core SDK initialization
│   ├── hooks/
│   │   ├── useWallet.ts        # Wallet connection
│   │   ├── useFhevm.ts         # SDK state management
│   │   ├── useEncrypt.ts       # Encryption operations
│   │   ├── useDecrypt.ts       # Decryption operations
│   │   └── index.ts            # Hook exports
│   └── components/             # UI components
├── test/                       # Contract tests
└── zama-protocol/              # Reference documentation
```

## Build Commands

### Smart Contract
```bash
npm install                     # Install dependencies
npm run compile                 # Compile contracts
npm run deploy:sepolia          # Deploy to Sepolia testnet
npm run test                    # Run tests
```

### Frontend
```bash
cd frontend
npm install
npm run dev                     # Start dev server (localhost:5173)
npm run build                   # Production build
```

### Environment Variables
```bash
# Root .env
SEPOLIA_RPC_URL=https://ethereum-sepolia-rpc.publicnode.com
PRIVATE_KEY=your_private_key

# Frontend .env
VITE_CONTRACT_ADDRESS=0x...
```

---

## Package Dependencies

### Solidity (package.json)
```json
{
  "dependencies": {
    "@fhevm/solidity": "^0.9.1"
  },
  "devDependencies": {
    "@fhevm/hardhat-plugin": "^0.3.0-1",
    "@openzeppelin/contracts": "^5.0.0",
    "hardhat": "^2.19.0",
    "typescript": "^5.3.0"
  }
}
```

### Frontend (frontend/package.json)
```json
{
  "dependencies": {
    "@zama-fhe/relayer-sdk": "^0.3.0-5",
    "ethers": "^6.9.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@vitejs/plugin-react": "^4.2.0",
    "typescript": "^5.3.0",
    "vite": "^5.0.0",
    "vite-plugin-node-polyfills": "^0.24.0",
    "vite-plugin-top-level-await": "^1.6.0",
    "vite-plugin-wasm": "^3.5.0"
  }
}
```

---

## Vite Configuration (Required)

```typescript
// frontend/vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { nodePolyfills } from "vite-plugin-node-polyfills";
import wasm from "vite-plugin-wasm";
import topLevelAwait from "vite-plugin-top-level-await";

export default defineConfig({
  plugins: [
    react(),
    wasm(),
    topLevelAwait(),
    nodePolyfills({
      include: ["buffer", "crypto", "stream", "util", "events"],
      globals: { Buffer: true, global: true, process: true },
    }),
  ],
  optimizeDeps: {
    include: ["keccak", "ethers", "fetch-retry"],
    exclude: ["@zama-fhe/relayer-sdk"],
    esbuildOptions: { target: "esnext" },
  },
  assetsInclude: ["**/*.wasm"],
  server: {
    headers: {
      "Cross-Origin-Opener-Policy": "same-origin",
      "Cross-Origin-Embedder-Policy": "require-corp",
    },
    fs: { allow: [".."] },
  },
  worker: {
    format: "es",
    plugins: () => [wasm(), topLevelAwait()],
  },
  build: { target: "esnext" },
});
```

---

## Key Concepts

### Encrypted Types
| Internal Type | External Type | Size |
|--------------|---------------|------|
| `ebool` | `externalEbool` | 1 bit |
| `euint8` | `externalEuint8` | 8 bits |
| `euint16` | `externalEuint16` | 16 bits |
| `euint32` | `externalEuint32` | 32 bits |
| `euint64` | `externalEuint64` | 64 bits |
| `euint128` | `externalEuint128` | 128 bits |
| `euint256` | `externalEuint256` | 256 bits |
| `eaddress` | `externalEaddress` | 160 bits |

### ACL (Access Control List)
Every encrypted value requires explicit permission grants:
```solidity
FHE.allowThis(handle);                    // Contract self-access
FHE.allow(handle, address);               // Permanent access
FHE.allowTransient(handle, address);      // Current transaction only
FHE.makePubliclyDecryptable(handle);      // Anyone can decrypt
```

### Control Flow
```solidity
// Use FHE.select for encrypted conditions (no if/else)
euint64 result = FHE.select(condition, trueValue, falseValue);
```

### Configuration Inheritance
```solidity
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract MyContract is ZamaEthereumConfig {
    // Your contract code
}
```

---

## Smart Contract Patterns

### Receiving Encrypted Input
```solidity
function myFunction(
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) external {
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);

    // Use the encrypted value...

    // Grant permissions
    FHE.allowThis(amount);
    FHE.allow(amount, msg.sender);
}
```

### Safe Arithmetic
```solidity
import {FHESafeMath} from "@openzeppelin/confidential-contracts/utils/FHESafeMath.sol";

(ebool success, euint64 result) = FHESafeMath.tryAdd(a, b);
euint64 finalValue = FHE.select(success, result, fallbackValue);
```

### Encrypted Comparison
```solidity
ebool isGreater = FHE.gt(a, b);
ebool isEqual = FHE.eq(a, b);
ebool isLessOrEqual = FHE.le(a, b);
```

---

## Frontend Architecture

### SDK Initialization Pattern
```typescript
// CRITICAL: Import from /web for browser, /node for Node.js
import { createInstance, SepoliaConfig, initSDK } from "@zama-fhe/relayer-sdk/web";

// 1. Initialize WASM first (required!)
await initSDK();

// 2. Create instance with network provider
const config = { ...SepoliaConfig, network: window.ethereum };
const instance = await createInstance(config);
```

### Modular Hook Architecture
```
useWallet() → useFhevm() → useEncrypt()/useDecrypt()
```

| Hook | Purpose |
|------|---------|
| `useWallet` | Wallet connection, chain detection, auto-switch to Sepolia |
| `useFhevm` | SDK initialization (idle → loading → ready states) |
| `useEncrypt` | Encryption operations |
| `useDecrypt` | Decryption with EIP-712 signatures |

### Initialization Flow
```typescript
// Initialize FHEVM only AFTER wallet connects
useEffect(() => {
  if (isConnected && fhevmStatus === "idle") {
    initializeFhevm();
  }
}, [isConnected, fhevmStatus, initializeFhevm]);
```

---

## Chain Configuration

| Network | Chain ID | Hex |
|---------|----------|-----|
| Ethereum Sepolia | 11155111 | 0xaa36a7 |
| Zama Gateway | 10901 | (internal) |

### Auto-Switch Network
```typescript
const SEPOLIA_CHAIN_ID_HEX = "0xaa36a7";

await window.ethereum.request({
  method: "wallet_switchEthereumChain",
  params: [{ chainId: SEPOLIA_CHAIN_ID_HEX }],
});
```

---

## SepoliaConfig (v0.3.0-5)

```typescript
const SepoliaConfig = {
  aclContractAddress: '0xf0Ffdc93b7E186bC2f8CB3dAA75D86d1930A433D',
  kmsContractAddress: '0xbE0E383937d564D7FF0BC3b46c51f0bF8d5C311A',
  inputVerifierContractAddress: '0xBBC1fFCdc7C316aAAd72E807D9b0272BE8F84DA0',
  verifyingContractAddressDecryption: '0x5D8BD78e2ea6bbE41f26dFe9fdaEAa349e077478',
  verifyingContractAddressInputVerification: '0x483b9dE06E4E4C7D35CCf5837A1668487406D955',
  chainId: 11155111,
  gatewayChainId: 10901,
  network: 'https://ethereum-sepolia-rpc.publicnode.com',
  relayerUrl: 'https://relayer.testnet.zama.org',
};
```

---

## Code Snippets

### Core FHEVM Module
```typescript
// src/core/fhevm.ts
import { createInstance, SepoliaConfig, initSDK } from "@zama-fhe/relayer-sdk/web";
import type { FhevmInstance } from "@zama-fhe/relayer-sdk/web";

let fheInstance: FhevmInstance | null = null;
let isInitialized = false;

export async function initializeFheInstance(): Promise<FhevmInstance> {
  if (fheInstance && isInitialized) return fheInstance;

  if (!window.ethereum) {
    throw new Error("Ethereum provider not found. Please install MetaMask.");
  }

  await initSDK();
  const config = { ...SepoliaConfig, network: window.ethereum };
  fheInstance = await createInstance(config);
  isInitialized = true;
  return fheInstance;
}

export function getFheInstance(): FhevmInstance | null {
  return fheInstance;
}
```

### Wallet Hook
```typescript
// src/hooks/useWallet.ts
import { useState, useCallback, useEffect } from "react";
import { ethers } from "ethers";

const SEPOLIA_CHAIN_ID = 11155111;
const SEPOLIA_CHAIN_ID_HEX = "0xaa36a7";

export function useWallet() {
  const [address, setAddress] = useState("");
  const [signer, setSigner] = useState<ethers.Signer | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const connect = useCallback(async () => {
    if (!window.ethereum) {
      setError("MetaMask not found");
      return;
    }

    try {
      await window.ethereum.request({ method: "eth_requestAccounts" });
      const chainIdHex = await window.ethereum.request({ method: "eth_chainId" });
      const chainId = parseInt(chainIdHex as string, 16);

      if (chainId !== SEPOLIA_CHAIN_ID) {
        await window.ethereum.request({
          method: "wallet_switchEthereumChain",
          params: [{ chainId: SEPOLIA_CHAIN_ID_HEX }],
        });
      }

      const provider = new ethers.BrowserProvider(window.ethereum);
      const signer = await provider.getSigner();
      setAddress(await signer.getAddress());
      setSigner(signer);
      setIsConnected(true);
      setError(null);
    } catch (err) {
      setError("Failed to connect wallet");
    }
  }, []);

  return { address, signer, isConnected, error, connect };
}
```

### FHEVM Hook
```typescript
// src/hooks/useFhevm.ts
import { useState, useCallback } from "react";
import { initializeFheInstance, getFheInstance } from "../core/fhevm";

export type FhevmStatus = "idle" | "loading" | "ready" | "error";

export function useFhevm() {
  const [status, setStatus] = useState<FhevmStatus>(
    getFheInstance() ? "ready" : "idle"
  );
  const [error, setError] = useState<string | null>(null);

  const initialize = useCallback(async () => {
    if (status === "loading" || status === "ready") return;

    setStatus("loading");
    try {
      await initializeFheInstance();
      setStatus("ready");
    } catch (err) {
      setError(err instanceof Error ? err.message : String(err));
      setStatus("error");
    }
  }, [status]);

  return { status, error, initialize, isInitialized: status === "ready" };
}
```

### Encrypt Hook
```typescript
// src/hooks/useEncrypt.ts
import { useState, useCallback } from "react";
import { getFheInstance } from "../core/fhevm";

export function useEncrypt() {
  const [isEncrypting, setIsEncrypting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const encrypt = useCallback(async (
    contractAddress: string,
    userAddress: string,
    value: bigint,
    type: "64" | "256" = "64"
  ) => {
    const instance = getFheInstance();
    if (!instance) {
      setError("FHEVM not initialized");
      return null;
    }

    setIsEncrypting(true);
    setError(null);

    try {
      const input = instance.createEncryptedInput(contractAddress, userAddress);
      if (type === "256") {
        input.add256(value);
      } else {
        input.add64(value);
      }
      return await input.encrypt();
    } catch (err) {
      setError(err instanceof Error ? err.message : String(err));
      return null;
    } finally {
      setIsEncrypting(false);
    }
  }, []);

  return { encrypt, isEncrypting, error };
}
```

### Decrypt Hook
```typescript
// src/hooks/useDecrypt.ts
import { useState, useCallback } from "react";
import { ethers } from "ethers";
import { getFheInstance } from "../core/fhevm";

export function useDecrypt() {
  const [isDecrypting, setIsDecrypting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const decrypt = useCallback(async (
    handles: { handle: string; contractAddress: string }[],
    signer: ethers.Signer,
    userAddress: string,
    contractAddresses: string[]
  ) => {
    const instance = getFheInstance();
    if (!instance) {
      setError("FHEVM not initialized");
      return new Map();
    }

    setIsDecrypting(true);
    setError(null);

    try {
      const keypair = instance.generateKeypair();
      const startTimestamp = Math.floor(Date.now() / 1000);
      const durationDays = 7;

      const eip712Message = instance.createEIP712(
        keypair.publicKey,
        contractAddresses,
        startTimestamp,
        durationDays
      );

      const signature = await signer.signTypedData(
        eip712Message.domain,
        eip712Message.types,
        eip712Message.message
      );

      const results = await instance.userDecrypt(
        handles,
        keypair.privateKey,
        keypair.publicKey,
        signature,
        contractAddresses,
        userAddress,
        startTimestamp,
        durationDays
      );

      return new Map(Object.entries(results));
    } catch (err) {
      setError(err instanceof Error ? err.message : String(err));
      return new Map();
    } finally {
      setIsDecrypting(false);
    }
  }, []);

  return { decrypt, isDecrypting, error };
}
```

---

## HCU (Homomorphic Computing Unit) Limits

Operations have computational costs. Per-transaction limits:
- **Global complexity**: 20,000,000 HCU
- **Sequential depth**: 5,000,000 HCU

Optimize by:
- Using smallest sufficient types (euint8 vs euint256)
- Preferring plaintext operands where possible
- Batching operations efficiently

---

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| `fetch-retry` export error | Add to `optimizeDeps.include` |
| WASM magic word error | Add `assetsInclude: ["**/*.wasm"]`, exclude SDK from optimizeDeps |
| Wrong chain detected | Use `eth_chainId` directly, implement auto-switch |
| FHEVM not ready | Initialize only after wallet connects |
| Cross-Origin error | Add COOP/COEP headers to Vite config |

---

## Reference Documentation

See `/zama-protocol/` folder:
- `01-solidity-workflow.md` - FHE operations, encrypted inputs, ACL patterns
- `02-relayer-sdk.md` - Frontend SDK integration (updated with latest patterns)
- `03-erc7984-confidential-tokens.md` - Confidential token standards
- `06-api-reference.md` - Quick reference cheat sheet

---

## External Resources

- [Zama FHEVM Documentation](https://docs.zama.ai/fhevm)
- [Relayer SDK Reference](https://docs.zama.ai/fhevm/frontend)
- [Contract Addresses](https://docs.zama.ai/fhevm/references/addresses)
- [fhevm-react-template](https://github.com/0xchriswilder/fhevm-react-template)

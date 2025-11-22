# Relayer SDK Integration Guide

## Overview

The Relayer SDK enables frontend applications to interact with FHEVM smart contracts without direct Gateway Chain involvement. All interactions with the Gateway chain are handled through HTTP calls to Zama's Relayer service.

## Installation

### NPM Package (Recommended)

```bash
npm install @zama-fhe/relayer-sdk@^0.3.0-5
```

### Required Vite Dependencies

```bash
npm install -D vite-plugin-wasm vite-plugin-top-level-await vite-plugin-node-polyfills
```

### CDN (Alternative)

```html
<script src="https://cdn.zama.org/relayer-sdk-js/0.3.0-5/relayer-sdk-js.umd.cjs"></script>
<script>
  const { initSDK, createInstance, SepoliaConfig } = window.RelayerSDK;

  async function init() {
    await initSDK();
    const config = { ...SepoliaConfig, network: window.ethereum };
    const instance = await createInstance(config);
  }
</script>
```

---

## SDK Initialization

The Relayer SDK requires:
1. **WASM initialization** via `initSDK()`
2. **Instance creation** with network provider

### Basic Setup (Sepolia) - Browser

```typescript
// IMPORTANT: Use /web import for browser environments
import { createInstance, SepoliaConfig, initSDK } from "@zama-fhe/relayer-sdk/web";

// Step 1: Initialize WASM modules (required!)
await initSDK();

// Step 2: Create instance with network provider
const config = {
  ...SepoliaConfig,
  network: window.ethereum  // Pass the wallet provider!
};
const instance = await createInstance(config);
```

### Basic Setup (Node.js)

```typescript
// Use /node import for Node.js environments
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/node";

const instance = await createInstance({
  ...SepoliaConfig,
  network: "https://ethereum-sepolia-rpc.publicnode.com"
});
```

### SepoliaConfig Values (v0.3.0-5)

```typescript
const SepoliaConfig = {
  aclContractAddress: '0xf0Ffdc93b7E186bC2f8CB3dAA75D86d1930A433D',
  kmsContractAddress: '0xbE0E383937d564D7FF0BC3b46c51f0bF8d5C311A',
  inputVerifierContractAddress: '0xBBC1fFCdc7C316aAAd72E807D9b0272BE8F84DA0',
  verifyingContractAddressDecryption: '0x5D8BD78e2ea6bbE41f26dFe9fdaEAa349e077478',
  verifyingContractAddressInputVerification: '0x483b9dE06E4E4C7D35CCf5837A1668487406D955',
  chainId: 11155111,           // Ethereum Sepolia
  gatewayChainId: 10901,       // Zama Gateway
  network: 'https://ethereum-sepolia-rpc.publicnode.com',
  relayerUrl: 'https://relayer.testnet.zama.org',
};
```

### Configuration Parameters

| Parameter | Description |
|-----------|-------------|
| `aclContractAddress` | ACL contract on FHEVM host chain |
| `kmsContractAddress` | KMS verifier contract on FHEVM host chain |
| `inputVerifierContractAddress` | Input verifier contract on FHEVM host chain |
| `verifyingContractAddressDecryption` | Decryption contract on Gateway chain |
| `verifyingContractAddressInputVerification` | Input verification contract on Gateway chain |
| `chainId` | FHEVM host chain ID (e.g., 11155111 for Sepolia) |
| `gatewayChainId` | Gateway chain ID (55815) |
| `network` | RPC provider URL or provider object |
| `relayerUrl` | Zama Relayer service endpoint |

### Pre-configured Networks

| Config | Network | Host Chain ID | Usage |
|--------|---------|---------------|-------|
| `SepoliaConfig` | Sepolia Testnet | 11155111 | Testing and development |

> **Note**: For the latest contract addresses, see the [Zama contract addresses page](https://docs.zama.ai/protocol/solidity-guides/smart-contract/configure/contract_addresses). For more information on the Relayer architecture, see the [Relayer documentation](https://docs.zama.ai/protocol/protocol/overview/relayer_oracle).

---

## Creating Encrypted Inputs

### Input Buffer Creation

```typescript
// Create input buffer for a specific contract and user
const inputBuffer = instance.createEncryptedInput(
  contractAddress,  // Target contract address
  userAddress       // User's wallet address
);
```

### Adding Values to Buffer

```typescript
// Add different encrypted types
inputBuffer.addBool(true);                    // ebool
inputBuffer.add8(255);                        // euint8
inputBuffer.add16(65535);                     // euint16
inputBuffer.add32(4294967295);                // euint32
inputBuffer.add64(BigInt("18446744073709551615")); // euint64
inputBuffer.add128(BigInt("..."));            // euint128
inputBuffer.add256(BigInt("..."));            // euint256
inputBuffer.addAddress("0x...");              // eaddress
```

### Encrypting and Submitting

```typescript
// Encrypt all values
const encryptedInput = await inputBuffer.encrypt();

// encryptedInput contains:
// - handles: Array of encrypted value handles
// - inputProof: Proof for validation

// Use in contract call
const tx = await contract.myFunction(
  encryptedInput.handles[0],  // First encrypted value
  encryptedInput.handles[1],  // Second encrypted value
  encryptedInput.inputProof   // Proof for all values
);
```

### Complete Input Example

```typescript
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk";
import { ethers } from "ethers";

async function depositConfidential(amount: bigint) {
  const provider = new ethers.BrowserProvider(window.ethereum);
  const signer = await provider.getSigner();
  const userAddress = await signer.getAddress();

  // Create instance with Sepolia configuration
  const instance = await createInstance(SepoliaConfig);

  // Create and encrypt input
  const input = instance.createEncryptedInput(
    CONTRACT_ADDRESS,
    userAddress
  );
  input.add64(amount);

  const encrypted = await input.encrypt();

  // Connect to contract and call function
  const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer);

  const tx = await contract.deposit(
    encrypted.handles[0],
    encrypted.inputProof
  );

  await tx.wait();
  console.log("Deposit successful!");
}
```

---

## User Decryption

User decryption allows users to decrypt their own encrypted values stored in contracts.

### Step 1: Generate Keypair

```typescript
// Generate a keypair for decryption
const keypair = instance.generateKeyPair();

// keypair contains:
// - publicKey: For creating decryption requests
// - privateKey: For decrypting responses (keep secret!)
```

### Step 2: Create EIP712 Message

```typescript
// Get the encrypted handle from the contract
const encryptedBalance = await contract.getBalance(userAddress);

// Create EIP712 typed data message
const eip712Message = instance.createEIP712Message(
  encryptedBalance,  // The handle to decrypt
  keypair.publicKey  // User's public key
);
```

### Step 3: Sign the Message

```typescript
const provider = new ethers.BrowserProvider(window.ethereum);
const signer = await provider.getSigner();

// Sign the EIP712 message
const signature = await signer.signTypedData(
  eip712Message.domain,
  eip712Message.types,
  eip712Message.message
);
```

### Step 4: Decrypt

```typescript
// Decrypt the value
const decryptedValue = await instance.userDecrypt(
  keypair,          // The keypair generated earlier
  signature,        // The EIP712 signature
  encryptedBalance  // The handle to decrypt
);

console.log("Your balance:", decryptedValue.toString());
```

### Complete User Decryption Example

```typescript
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk";
import { ethers } from "ethers";

async function getMyBalance() {
  const provider = new ethers.BrowserProvider(window.ethereum);
  const signer = await provider.getSigner();
  const userAddress = await signer.getAddress();

  // Create instance
  const instance = await createInstance(SepoliaConfig);

  // Get encrypted balance from contract
  const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer);
  const encryptedBalance = await contract.confidentialBalanceOf(userAddress);

  // Generate keypair
  const keypair = instance.generateKeyPair();

  // Create and sign EIP712 message
  const eip712Message = instance.createEIP712Message(
    encryptedBalance,
    keypair.publicKey
  );

  const signature = await signer.signTypedData(
    eip712Message.domain,
    eip712Message.types,
    eip712Message.message
  );

  // Decrypt
  const balance = await instance.userDecrypt(
    keypair,
    signature,
    encryptedBalance
  );

  return balance;
}
```

---

## Public Decryption

For values marked as publicly decryptable (`FHE.makePubliclyDecryptable()`), anyone can decrypt without signatures.

```typescript
// Single handle
const decryptedValue = await instance.publicDecrypt(handle);

// Multiple handles
const handles = [
  "0x830a61b343d2f3de67ec59cb18961fd086085c1c73ff0000000000aa36a70000",
  "0x98ee526413903d4613feedb9c8fa44fe3f4ed0dd00ff0000000000aa36a70400"
];

const values = await instance.publicDecrypt(handles);
// Returns: Map<handle, decryptedValue>
```

### Return Types

| Encrypted Type | Decrypted Type |
|----------------|----------------|
| `ebool` | `boolean` |
| `euint8` - `euint256` | `BigInt` |
| `eaddress` | `string` (0x-prefixed) |

---

## Web Application Integration

### Vite Configuration (Required)

```typescript
// vite.config.ts
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
  build: { target: "esnext" },
});
```

### React Modular Hooks Pattern (Recommended)

#### Core FHEVM Module

```typescript
// src/core/fhevm.ts
import { createInstance, SepoliaConfig, initSDK } from "@zama-fhe/relayer-sdk/web";
import type { FhevmInstance } from "@zama-fhe/relayer-sdk/web";

let fheInstance: FhevmInstance | null = null;

export async function initializeFheInstance(): Promise<FhevmInstance> {
  if (fheInstance) return fheInstance;

  if (!window.ethereum) {
    throw new Error("Ethereum provider not found");
  }

  await initSDK();
  const config = { ...SepoliaConfig, network: window.ethereum };
  fheInstance = await createInstance(config);
  return fheInstance;
}

export function getFheInstance(): FhevmInstance | null {
  return fheInstance;
}
```

#### Wallet Hook

```typescript
// src/hooks/useWallet.ts
import { useState, useCallback } from "react";
import { ethers } from "ethers";

const SEPOLIA_CHAIN_ID = 11155111;
const SEPOLIA_CHAIN_ID_HEX = "0xaa36a7";

export function useWallet() {
  const [address, setAddress] = useState("");
  const [signer, setSigner] = useState<ethers.Signer | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  const connect = useCallback(async () => {
    if (!window.ethereum) throw new Error("MetaMask not found");

    await window.ethereum.request({ method: "eth_requestAccounts" });
    const chainIdHex = await window.ethereum.request({ method: "eth_chainId" });
    const chainId = parseInt(chainIdHex as string, 16);

    // Auto-switch to Sepolia if needed
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
  }, []);

  return { address, signer, isConnected, connect };
}
```

#### FHEVM Hook

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

#### Encrypt Hook

```typescript
// src/hooks/useEncrypt.ts
import { useState, useCallback } from "react";
import { getFheInstance } from "../core/fhevm";

export function useEncrypt() {
  const [isEncrypting, setIsEncrypting] = useState(false);

  const encrypt256 = useCallback(async (
    contractAddress: string,
    userAddress: string,
    value: bigint
  ) => {
    const instance = getFheInstance();
    if (!instance) throw new Error("FHEVM not initialized");

    setIsEncrypting(true);
    try {
      const input = instance.createEncryptedInput(contractAddress, userAddress);
      input.add256(value);
      return await input.encrypt();
    } finally {
      setIsEncrypting(false);
    }
  }, []);

  return { encrypt256, isEncrypting };
}
```

#### Usage in App

```typescript
// src/App.tsx
import { useEffect } from "react";
import { useWallet } from "./hooks/useWallet";
import { useFhevm } from "./hooks/useFhevm";
import { useEncrypt } from "./hooks/useEncrypt";

function App() {
  const { address, isConnected, connect } = useWallet();
  const { status, initialize, isInitialized } = useFhevm();
  const { encrypt256, isEncrypting } = useEncrypt();

  // Initialize FHEVM after wallet connects
  useEffect(() => {
    if (isConnected && status === "idle") {
      initialize();
    }
  }, [isConnected, status, initialize]);

  const handleSend = async () => {
    if (!isInitialized) return;
    const encrypted = await encrypt256(CONTRACT_ADDRESS, address, BigInt(100));
    // Use encrypted.handles[0] and encrypted.inputProof in contract call
  };

  return (
    <div>
      {!isConnected ? (
        <button onClick={connect}>Connect Wallet</button>
      ) : (
        <div>
          <p>Connected: {address}</p>
          <p>FHEVM: {status}</p>
          {isInitialized && <button onClick={handleSend}>Send</button>}
        </div>
      )}
    </div>
  );
}
```

### Vue Example

```typescript
import { ref, onMounted } from "vue";
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk";

export function useRelayerSDK() {
  const instance = ref(null);
  const isReady = ref(false);

  onMounted(async () => {
    instance.value = await createInstance(SepoliaConfig);
    isReady.value = true;
  });

  const encrypt = async (contractAddress, userAddress, values) => {
    if (!instance.value) throw new Error("SDK not initialized");

    const input = instance.value.createEncryptedInput(
      contractAddress,
      userAddress
    );

    for (const { type, value } of values) {
      switch (type) {
        case "uint64": input.add64(value); break;
        case "uint32": input.add32(value); break;
        case "bool": input.addBool(value); break;
        // ... other types
      }
    }

    return await input.encrypt();
  };

  return { instance, isReady, encrypt };
}
```

---

## Error Handling

```typescript
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk";

try {
  const instance = await createInstance(SepoliaConfig);

  const input = instance.createEncryptedInput(contractAddress, userAddress);
  input.add64(amount);
  const encrypted = await input.encrypt();
} catch (error) {
  if (error.code) {
    switch (error.code) {
      case "NETWORK_ERROR":
        console.error("Failed to connect to relayer:", error.message);
        break;
      case "INVALID_PROOF":
        console.error("Input proof validation failed");
        break;
      case "ACL_DENIED":
        console.error("No permission to decrypt this value");
        break;
      default:
        console.error("FHEVM error:", error.message);
    }
  } else {
    throw error;
  }
}
```

---

## SDK API Reference

### Instance Methods

| Method | Description |
|--------|-------------|
| `createEncryptedInput(contract, user)` | Create input buffer for encryption |
| `generateKeyPair()` | Generate keypair for decryption |
| `createEIP712Message(handle, publicKey)` | Create EIP712 message for signing |
| `userDecrypt(keypair, signature, handle)` | Decrypt value with user signature |
| `publicDecrypt(handle \| handles[])` | Decrypt publicly decryptable values |

### Input Buffer Methods

| Method | Type | Description |
|--------|------|-------------|
| `addBool(value)` | `ebool` | Add encrypted boolean |
| `add8(value)` | `euint8` | Add encrypted uint8 |
| `add16(value)` | `euint16` | Add encrypted uint16 |
| `add32(value)` | `euint32` | Add encrypted uint32 |
| `add64(value)` | `euint64` | Add encrypted uint64 |
| `add128(value)` | `euint128` | Add encrypted uint128 |
| `add256(value)` | `euint256` | Add encrypted uint256 |
| `addAddress(value)` | `eaddress` | Add encrypted address |
| `encrypt()` | - | Encrypt all values and return handles + proof |

---

## Troubleshooting

### Common Errors

#### `fetch-retry` export error
```
The requested module 'fetch-retry' does not provide an export named 'default'
```
**Solution**: Add `fetch-retry` to `optimizeDeps.include` in vite.config.ts

#### WASM loading error
```
WebAssembly.instantiate(): expected magic word 00 61 73 6d, found 3c 21 44 4f
```
**Solution**:
1. Add `assetsInclude: ["**/*.wasm"]` to vite.config.ts
2. Add `@zama-fhe/relayer-sdk` to `optimizeDeps.exclude`
3. Clear Vite cache: `rm -rf node_modules/.vite`

#### "Please switch to Sepolia testnet"
**Cause**: Wallet is on wrong network (e.g., Mainnet instead of Sepolia)
**Solution**: Implement auto-switch using `wallet_switchEthereumChain`:
```typescript
await window.ethereum.request({
  method: "wallet_switchEthereumChain",
  params: [{ chainId: "0xaa36a7" }],  // Sepolia
});
```

#### FHEVM not initialized
**Cause**: SDK initialization happens before wallet connection
**Solution**: Initialize FHEVM only after wallet connects:
```typescript
useEffect(() => {
  if (isConnected && fhevmStatus === "idle") {
    initializeFhevm();
  }
}, [isConnected, fhevmStatus]);
```

#### Cross-Origin headers error
**Solution**: Add required headers to vite.config.ts:
```typescript
server: {
  headers: {
    "Cross-Origin-Opener-Policy": "same-origin",
    "Cross-Origin-Embedder-Policy": "require-corp",
  },
}
```

### Best Practices

1. **Always import from `/web` or `/node`** - Not from the package root
2. **Call `initSDK()` before `createInstance()`** - WASM must initialize first
3. **Pass `window.ethereum` as network** - Required for proper chain communication
4. **Initialize FHEVM after wallet connects** - Never before
5. **Use modular hooks** - Separate concerns for maintainability
6. **Implement chain auto-switching** - Better UX for users on wrong network

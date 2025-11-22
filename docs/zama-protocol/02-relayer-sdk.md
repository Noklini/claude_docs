# Relayer SDK Integration Guide

## Overview

The Relayer SDK enables frontend applications to interact with FHEVM smart contracts without direct Gateway Chain involvement. All interactions with the Gateway chain are handled through HTTP calls to Zama's Relayer service.

## Installation

### NPM Package (Recommended)

```bash
npm install @zama-fhe/relayer-sdk
```

### ESM CDN

```html
<script type="module">
  import { createInstance, SepoliaConfig } from
    'https://cdn.zama.ai/relayer-sdk/v0.x/bundle.esm.min.js';

  const instance = await createInstance(SepoliaConfig);
</script>
```

### UMD CDN (for SSR/legacy)

```html
<script src="https://cdn.zama.ai/relayer-sdk/v0.x/bundle.umd.min.js"></script>
<script>
  const { createInstance, SepoliaConfig } = window.fhevmjs;

  async function init() {
    const instance = await createInstance(SepoliaConfig);
  }
</script>
```

---

## SDK Initialization

The Relayer SDK requires instantiation of an `FhevmInstance` which holds all configuration and methods needed to interact with FHEVM.

### Basic Setup (Sepolia)

```typescript
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk";

// Create SDK instance using pre-configured Sepolia settings
const instance = await createInstance(SepoliaConfig);
```

### Custom Configuration

```typescript
import { createInstance } from "@zama-fhe/relayer-sdk";

const instance = await createInstance({
  // FHEVM Host Chain contracts (Sepolia)
  aclContractAddress: "0x687820221192C5B662b25367F70076A37bc79b6c",
  kmsContractAddress: "0x1364cBBf2cDF5032C47d8226a6f6FBD2AFCDacAC",
  inputVerifierContractAddress: "0xbc91f3daD1A5F19F8390c400196e58073B6a0BC4",

  // Gateway Chain contracts
  verifyingContractAddressDecryption: "0xb6E160B1ff80D67Bfe90A85eE06Ce0A2613607D1",
  verifyingContractAddressInputVerification: "0x7048C39f048125eDa9d678AEbaDfB22F7900a29F",

  // Chain IDs
  chainId: 11155111,        // FHEVM Host chain (Sepolia)
  gatewayChainId: 55815,    // Gateway chain

  // Optional: RPC provider (URL string or provider object)
  network: "https://eth-sepolia.public.blastapi.io",

  // Relayer URL
  relayerUrl: "https://relayer.testnet.zama.cloud",
});
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

### React Example

```typescript
import { useState, useEffect } from "react";
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk";
import { ethers } from "ethers";

function useConfidentialToken(contractAddress: string) {
  const [instance, setInstance] = useState(null);
  const [balance, setBalance] = useState<bigint | null>(null);

  useEffect(() => {
    async function init() {
      const inst = await createInstance(SepoliaConfig);
      setInstance(inst);
    }
    init();
  }, []);

  const transfer = async (to: string, amount: bigint) => {
    if (!instance) return;

    const provider = new ethers.BrowserProvider(window.ethereum);
    const signer = await provider.getSigner();
    const userAddress = await signer.getAddress();

    // Create encrypted input
    const input = instance.createEncryptedInput(contractAddress, userAddress);
    input.add64(amount);
    const encrypted = await input.encrypt();

    // Execute transfer
    const contract = new ethers.Contract(contractAddress, ABI, signer);
    const tx = await contract.confidentialTransfer(
      to,
      encrypted.handles[0],
      encrypted.inputProof
    );
    await tx.wait();
  };

  const refreshBalance = async () => {
    if (!instance) return;

    const provider = new ethers.BrowserProvider(window.ethereum);
    const signer = await provider.getSigner();
    const userAddress = await signer.getAddress();

    const contract = new ethers.Contract(contractAddress, ABI, signer);
    const encryptedBalance = await contract.confidentialBalanceOf(userAddress);

    // Decrypt balance
    const keypair = instance.generateKeyPair();
    const eip712Message = instance.createEIP712Message(
      encryptedBalance,
      keypair.publicKey
    );

    const signature = await signer.signTypedData(
      eip712Message.domain,
      eip712Message.types,
      eip712Message.message
    );

    const decrypted = await instance.userDecrypt(
      keypair,
      signature,
      encryptedBalance
    );

    setBalance(decrypted);
  };

  return { balance, transfer, refreshBalance };
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

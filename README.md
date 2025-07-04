# Quantum Chain SDK

A node.js SDK for interacting with the Quantum Chain network using WebAssembly for cryptographic operations.

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
- [Examples](#examples)
- [Configuration](#configuration)
- [Error Handling](#error-handling)
- [Contributing](#contributing)
- [License](#license)

## Installation

```bash
npm install @quantum_chain/sdk
npm install bip39  # Required for mnemonic functionality
```

## Quick Start

### Complete Transaction Example with Mnemonic

```javascript
const path = require('path');
const fs = require('fs');
const QuantumChainSDK = require('@quantum_chain/sdk');
const bip39 = require('bip39');

async function main() {
    // Define the mnemonic phrase (use your own secure mnemonic)
    const mnemonic = 'abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about';
    
    // Validate the mnemonic
    if (!bip39.validateMnemonic(mnemonic)) {
        throw new Error('Invalid mnemonic phrase');
    }
    
    // Generate seed from mnemonic (BIP39 standard)
    const seed = bip39.mnemonicToSeedSync(mnemonic);
    console.log('BIP39 Seed generated');
    
    // Extract first 32 bytes for account derivation
    const first32Bytes = seed.slice(0, 32);
    const seedHex = first32Bytes.toString('hex');
    console.log('Seed length:', first32Bytes.length, 'bytes');
    
    // Initialize the SDK
    const sdk = new QuantumChainSDK({ 
        rpcUrl: 'https://rpc.quantumchain.network' // Replace with your RPC URL
    });
    await sdk.initialize();

    // Load account from seed
    const account = await sdk.loadAccountFromSeed(seedHex);
    console.log('Account loaded from seed');

    // Get the account address
    const address = await sdk.getAddress();
    console.log('Account Address:', address);

    // Prepare transaction details
    const nonceHex = await sdk.getTransactionCount(address);
    const nonce = parseInt(nonceHex, 16);

    const gasPriceHex = await sdk.getGasPrice();
    const gasPrice = parseInt(gasPriceHex, 16);

    const toAddress = '0x742d35cc6634c0532925a3b8d0c3b3c3b3c3b3c3'; // Replace with recipient
    const amount = '1000000000000000000'; // 1 QC in Qwei

    const gasLimitHex = await sdk.estimateGas(address, toAddress, '0x' + BigInt(amount).toString(16));
    const gasLimit = parseInt(gasLimitHex, 16);

    // Sign the transaction
    const signedTx = await sdk.signTransaction(nonce, gasPrice, gasLimit, toAddress, amount);
    console.log('Transaction signed successfully');

    // Send the signed transaction
    const txHash = await sdk.sendRawTransaction(signedTx);
    console.log('Transaction Hash:', txHash);

    // Wait for the transaction receipt
    setTimeout(async () => {
        try {
            const receipt = await sdk.getTransactionReceipt(txHash);
            console.log('Transaction confirmed:', receipt.status);
        } catch (error) {
            console.log('Transaction still pending or failed');
        }
    }, 30000);
}

main().catch(console.error);
```

## API Reference

### Constructor

#### `new QuantumChainSDK(options)`

Creates a new instance of the Quantum Chain SDK.

**Parameters:**

- `options` (Object, optional): Configuration options
  - `rpcUrl` (string, optional): The Quantum Chain node RPC URL. Default: `'http://localhost:8545'`
  - `wasmPath` (string, optional): Path to the WebAssembly file. Default: `'../wasm/qc.wasm'`

### Methods

#### `initialize()`

Initializes the SDK by loading the WebAssembly module. Must be called before using any other methods.

**Returns:** `Promise<void>`

**Example:**

```javascript
await sdk.initialize();
```

#### `createAccount(passphrase)`

Creates a new Quantum Chain account with the specified passphrase.

**Parameters:**

- `passphrase` (string): The passphrase to secure the new account

**Returns:** `Promise<any>` - The newly created account object

**Example:**

```javascript
const account = await sdk.createAccount("my-secure-passphrase");
```

#### `loadAccount(keyfile, passphrase)`

Loads and decrypts an existing Quantum Chain account from a keyfile.

**Parameters:**

- `keyfile` (string): The JSON content of the keyfile
- `passphrase` (string): The passphrase to decrypt the keyfile

**Returns:** `Promise<any>` - The loaded account object

**Example:**

```javascript
const keyfileContent = fs.readFileSync("path/to/keyfile.json", "utf8");
const account = await sdk.loadAccount(keyfileContent, "my-passphrase");
```

#### `loadAccountFromSeed(seedHex)`

Loads a Quantum Chain account from a hexadecimal seed string. The seed should be exactly 32 bytes (64 hex characters).

**Parameters:**

- `seedHex` (string): The seed as a hexadecimal string (64 characters)

**Returns:** `Promise<any>` - The loaded account object

**Example:**

```javascript
const seedHex = "a1b2c3d4e5f6789012345678901234567890123456789012345678901234567890";
const account = await sdk.loadAccountFromSeed(seedHex);
```

#### `loadAccountFromMnemonic(mnemonic)`

Loads a Quantum Chain account from a BIP39 mnemonic phrase. The mnemonic is converted to a seed using the BIP39 standard.

**Parameters:**

- `mnemonic` (string): A valid BIP39 mnemonic phrase (12, 15, 18, 21, or 24 words)

**Returns:** `Promise<any>` - The loaded account object

**Example:**

```javascript
const mnemonic = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about";
const account = await sdk.loadAccountFromMnemonic(mnemonic);
```

**Note:** This method requires the `bip39` package to be installed:
```bash
npm install bip39
```

#### `getAddress()`

Retrieves the Quantum Chain address from the currently loaded account.

**Returns:** `Promise<string>` - The Quantum Chain address

**Example:**

```javascript
const address = await sdk.getAddress();
console.log("Address:", address);
```

#### `signTransaction(nonce, gasPrice, gasLimit, toAddress, amount)`

Signs a transaction using the loaded account.

**Parameters:**

- `nonce` (number): The transaction nonce
- `gasPrice` (number): The gas price in Qwei
- `gasLimit` (number): The gas limit
- `toAddress` (string): The recipient's Quantum Chain address
- `amount` (string): The amount in Qwei as a decimal string

**Returns:** `Promise<string>` - The signed transaction as a hex string

**Example:**

```javascript
const signedTx = await sdk.signTransaction(
  1, // nonce
  20000000000, // gasPrice (20 Gwei)
  21000, // gasLimit
  "0x742d35cc6634c0532925a3b8d0c3b3c3b3c3b3c3", // toAddress
  "1000000000000000000" // amount (1 QC)
);
```

#### `sendRawTransaction(signedTx)`

Sends a signed transaction to the Quantum Chain network.

**Parameters:**

- `signedTx` (string): The signed transaction hex string

**Returns:** `Promise<string>` - The transaction hash

**Example:**

```javascript
const txHash = await sdk.sendRawTransaction(signedTx);
console.log("Transaction hash:", txHash);
```

#### `getTransactionReceipt(txHash)`

Retrieves the transaction receipt for a given transaction hash.

**Parameters:**

- `txHash` (string): The transaction hash

**Returns:** `Promise<any>` - The transaction receipt object

**Example:**

```javascript
const receipt = await sdk.getTransactionReceipt(txHash);
console.log("Transaction receipt:", receipt);
```

#### `getTransactionCount(address)`

Retrieves the transaction count (nonce) for a given address.

**Parameters:**

- `address` (string): The Quantum Chain address

**Returns:** `Promise<string>` - The nonce as a hexadecimal string

**Example:**

```javascript
const nonce = await sdk.getTransactionCount(
  "0x742d35cc6634c0532925a3b8d0c3b3c3b3c3b3c3"
);
console.log("Nonce:", parseInt(nonce, 16));
```

#### `getGasPrice()`

Retrieves the current gas price from the network.

**Returns:** `Promise<string>` - The gas price as a hexadecimal string

**Example:**

```javascript
const gasPrice = await sdk.getGasPrice();
console.log("Gas price:", parseInt(gasPrice, 16));
```

#### `estimateGas(from, to, value)`

Estimates the gas required for a transaction.

**Parameters:**

- `from` (string): The sender address
- `to` (string): The recipient address
- `value` (string): The amount in Qwei as a hexadecimal string

**Returns:** `Promise<string>` - The estimated gas as a hexadecimal string

**Example:**

```javascript
const estimatedGas = await sdk.estimateGas(
  "0x742d35cc6634c0532925a3b8d0c3b3c3b3c3b3c3",
  "0x123d35cc6634c0532925a3b8d0c3b3c3b3c3b3c3",
  "0xde0b6b3a7640000" // 1 QC in hex
);
```

## Examples

### Complete Transaction Example

```javascript
const QuantumChainSDK = require("@quantum_chain/sdk");
const fs = require("fs");

async function sendTransaction() {
  try {
    // Initialize SDK
    const sdk = new QuantumChainSDK({
      rpcUrl: "https://rpc.quantumchain.network",
    });
    await sdk.initialize();

    // Load account from keyfile
    const keyfileContent = fs.readFileSync("./keyfile.json", "utf8");
    await sdk.loadAccount(keyfileContent, "your-passphrase");

    // Get account address
    const fromAddress = await sdk.getAddress();
    console.log("From address:", fromAddress);

    // Transaction parameters
    const toAddress = "0x742d35cc6634c0532925a3b8d0c3b3c3b3c3b3c3";
    const amount = "1000000000000000000"; // 1 QC

    // Get transaction details
    const nonce = parseInt(await sdk.getTransactionCount(fromAddress), 16);
    const gasPrice = parseInt(await sdk.getGasPrice(), 16);
    const gasLimit = parseInt(
      await sdk.estimateGas(fromAddress, toAddress, "0xde0b6b3a7640000"),
      16
    );

    console.log("Transaction details:", { nonce, gasPrice, gasLimit });

    // Sign transaction
    const signedTx = await sdk.signTransaction(
      nonce,
      gasPrice,
      gasLimit,
      toAddress,
      amount
    );

    // Send transaction
    const txHash = await sdk.sendRawTransaction(signedTx);
    console.log("Transaction sent! Hash:", txHash);

    // Wait for confirmation and get receipt
    setTimeout(async () => {
      try {
        const receipt = await sdk.getTransactionReceipt(txHash);
        console.log("Transaction receipt:", receipt);
      } catch (error) {
        console.log("Transaction still pending or failed");
      }
    }, 5000);
  } catch (error) {
    console.error("Error sending transaction:", error);
  }
}

sendTransaction();
```

### Creating and Using a New Account

```javascript
const QuantumChainSDK = require("@quantum_chain/sdk");
const fs = require("fs");

async function createAndUseAccount() {
  try {
    const sdk = new QuantumChainSDK();
    await sdk.initialize();

    // Create new account
    const account = await sdk.createAccount("secure-passphrase-123");
    console.log("New account created:", account);

    // Get address
    const address = await sdk.getAddress();
    console.log("Account address:", address);

    // Save account to file (implement your own secure storage)
    // fs.writeFileSync('./new-keyfile.json', JSON.stringify(account));
  } catch (error) {
    console.error("Error creating account:", error);
  }
}

createAndUseAccount();
```

### Working with Mnemonic Phrases

```javascript
const QuantumChainSDK = require("@quantum_chain/sdk");
const bip39 = require("bip39");

async function workWithMnemonic() {
  try {
    const sdk = new QuantumChainSDK({
      rpcUrl: "https://rpc.quantumchain.network"
    });
    await sdk.initialize();

    // Generate a new mnemonic phrase
    const mnemonic = bip39.generateMnemonic();
    console.log("Generated mnemonic:", mnemonic);

    // Validate the mnemonic
    const isValid = bip39.validateMnemonic(mnemonic);
    console.log("Mnemonic is valid:", isValid);

    // Load account directly from mnemonic
    const account = await sdk.loadAccountFromMnemonic(mnemonic);
    console.log("Account loaded from mnemonic");

    // Get the address
    const address = await sdk.getAddress();
    console.log("Account address:", address);

    // Alternative: Convert mnemonic to seed manually
    const seed = bip39.mnemonicToSeedSync(mnemonic);
    const seedHex = seed.slice(0, 32).toString('hex');
    console.log("Seed (32 bytes hex):", seedHex);

    // Load account from seed
    const accountFromSeed = await sdk.loadAccountFromSeed(seedHex);
    console.log("Account loaded from seed");
  } catch (error) {
    console.error("Error working with mnemonic:", error);
  }
}

workWithMnemonic();
```

### Working with Custom Seeds

```javascript
const QuantumChainSDK = require("@quantum_chain/sdk");
const crypto = require("crypto");

async function workWithCustomSeed() {
  try {
    const sdk = new QuantumChainSDK();
    await sdk.initialize();

    // Generate a random 32-byte seed
    const randomSeed = crypto.randomBytes(32);
    const seedHex = randomSeed.toString('hex');
    console.log("Generated seed:", seedHex);

    // Load account from the seed
    const account = await sdk.loadAccountFromSeed(seedHex);
    console.log("Account loaded from custom seed");

    const address = await sdk.getAddress();
    console.log("Account address:", address);

    // Store the seed securely for future use
    // WARNING: Store seeds securely! This is just an example
    // fs.writeFileSync('./seed.txt', seedHex);
  } catch (error) {
    console.error("Error working with custom seed:", error);
  }
}

workWithCustomSeed();
```

## Configuration

### RPC Configuration

The SDK supports custom RPC endpoints. You can configure the RPC URL when initializing the SDK:

```javascript
const sdk = new QuantumChainSDK({
  rpcUrl: "https://your-quantum-chain-node.com:8545",
});
```

### WebAssembly Path

If your WASM files are located in a different directory, you can specify the path:

```javascript
const sdk = new QuantumChainSDK({
  wasmPath: "./custom/path/to/qc.wasm",
});
```

## Error Handling

The SDK throws errors for various conditions. Always wrap SDK calls in try-catch blocks:

```javascript
try {
  await sdk.initialize();
  const account = await sdk.loadAccount(keyfile, passphrase);
} catch (error) {
  if (error.message.includes("WASM module not initialized")) {
    console.error("Initialize the SDK first");
  } else if (error.message.includes("RPC Error")) {
    console.error("Network error:", error.message);
  } else {
    console.error("Unexpected error:", error);
  }
}
```

## Requirements

- Node.js 14.0.0 or higher
- The WebAssembly files (`qc.wasm` and `wasm_exec.js`) must be available
- Access to a Quantum Chain RPC endpoint
- `bip39` package (for mnemonic functionality): `npm install bip39`

## File Structure

```
quantum-sdk/
├── lib/
│   ├── index.js      # Main SDK class
│   ├── account.js    # Account management functions
│   ├── transaction.js # Transaction signing functions
│   ├── rpc.js        # RPC communication functions
│   └── wasm.js       # WebAssembly loading functions
├── wasm/
│   ├── qc.wasm       # Quantum Chain WebAssembly module
│   └── wasm_exec.js  # Go WebAssembly runtime
├── package.json
└── README.md
```

## Contributing

This SDK is proprietary software. For questions or issues, please contact the development team.

## License

Proprietary - See license terms in your agreement with Quantum Chain PTE LTD.

## Support

- Homepage: https://www.quantumcha.in
- Issues: https://github.com/Quantum-Chain-PTE-LTD/quantum-sdk/issues

---

**Note:** This SDK uses WebAssembly for cryptographic operations. Ensure your environment supports WebAssembly and that the required WASM files are properly deployed with your application.

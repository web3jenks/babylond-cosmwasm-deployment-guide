# Babylon Smart Contract Deployment Guide

## Introduction
This guide walks you through deploying a smart contract on the Babylon blockchain. The example contract we'll deploy is a storage contract that allows saving data with Bitcoin timestamps and verifying Bitcoin finalization status.

## Prerequisites

Before starting, ensure you have installed:
- [Rust](https://www.rust-lang.org/tools/install) - Required for building the CosmWASM contract and CLI tool
- [Docker](https://docs.docker.com/get-docker/) - Used for contract optimization

## Detailed Deployment Steps

### 1. Repository Setup
First, clone the repository containing the Babylon smart contract code. The repository includes submodules for Babylon core and contract dependencies.

Update the submodules:
```bash
git submodule update --init --recursive
```
This command initializes and fetches all necessary submodule code: `babyon` binary and `storage_contract`. 

### 2. Babylond CLI Installation
The Babylond CLI is your primary tool for interacting with the Babylon blockchain.

Verify your Rust installation first:
```bash
rustc --version
# Expected output similar to: rustc 1.81.0 (2dbb1af80 2024-08-20)
```

Build and install the CLI:
```bash
cd babylon
make install
```

Verify the installation:
```bash
babylond version
# Should output a version hash similar to the follwoing: 
main-112821536b0ada40aa29e34b53206f56c61bf631
```

### 3. Wallet Management

#### Create a New Wallet
Create a test wallet using the local keyring:
```bash
babylond keys add test-key --keyring-backend=test
```
This command:
- Creates a new key pair
- Stores it in the test keyring
- Outputs the address and recovery phrase
- IMPORTANT: Save the mnemonic phrase securely for recovery

Verify the wallet:
```bash
babylond keys list --keyring-backend=test
```

#### Fund Your Wallet
You need test tokens (tBBN) to deploy contracts. Get them through:

Option 1: L2Scan Faucet
1. Visit [L2Scan Faucet](https://babylon-testnet.l2scan.co/faucet)
2. Create an account
3. Request test tokens

Option 2: Command Line
```bash
curl https://faucet.testnet.babylonlabs.io/claim \
-H "Content-Type: multipart/form-data" \
-d "{\"address\":\"$(babylond keys show $key -a --keyring-backend=test)\"}"
```

Verify your balance:
```bash
babylond query bank balances $(babylond keys show $key -a --keyring-backend=test) --node=$nodeUrl
```
Expected output should show some tBBN tokens.

### 4. Environment Configuration

Load network-specific variables:
```bash
# For Phase 2 testnet (Babylon chain)
source env-phase2-testnet.sh

# OR for Phase 3 devnet (multi-staking)
source env-phase3-devnet.sh
```

These files set crucial variables:
- `$homeDir`: Babylond configuration directory
- `$chainId`: Network identifier
- `$feeToken`: Token used for transaction fees
- `$key`: Your wallet name
- `$nodeUrl`: RPC endpoint
- `$apiUrl`: REST API endpoint

Verify the configuration:
```bash
echo $homeDir, $chainId, $feeToken, $key, $nodeUrl, $apiUrl
```

### 5. Contract Building

#### Verify Docker
The build process uses Docker for consistent environments:
```bash
docker run hello-world
```

#### Build Contract
Navigate to the contract directory and build:
```bash
cd storage-contract
cargo run-script optimize
```
This creates an optimized WASM file in the `artifacts` directory.

### 6. Contract Deployment Process

#### Step 1: Store Contract Code
Upload the WASM file to the blockchain:
```bash
babylond tx wasm store ./artifacts/storage_contract-aarch64.wasm \
--from=$key \
--gas=auto \
--gas-prices=1$feeToken \
--gas-adjustment=1.3 \
--chain-id="$chainId" \
-b=sync \
--yes \
--keyring-backend=test \
--log_format=json \
--home=$homeDir \
--node=$nodeUrl
```

Key parameters explained:
- `--gas=auto`: Automatically estimates required gas
- `--gas-adjustment=1.3`: Adds 30% to estimated gas for safety
- `-b=sync`: Waits for transaction to be broadcast
- `--yes`: Automatically confirms the transaction

#### Step 2: Get Contract Code ID
After storing, get the unique code identifier:
```bash
codeID=$(babylond query wasm list-code \
--node $nodeUrl \
-o json \
| jq \
--arg ADDR "$(babylond keys show $key -a --keyring-backend=test)" \
'.code_infos[] | select(.creator==$ADDR).code_id | tonumber')
```
This command filters contracts against your wallet address to find the one you just uploaded.

#### Step 3: Instantiate Contract
Create a new instance of the contrac using `tx wasm instantiate` command:
```bash
babylond tx wasm instantiate $codeID '{}' \
--from=$key \
--no-admin \
--label="storage_contract" \
--gas=auto \
--gas-prices=1$feeToken \
--gas-adjustment=1.3 \
--chain-id="$chainId" \
-b=sync \
--yes \
--keyring-backend=test \
--log_format=json \
--home=$homeDir \
--node=$nodeUrl
```

Parameters explained:
- `'{}'`: Empty initialization parameters for the storage_contract
- `--no-admin`: Creates contract without admin privileges
- `--label`: Human-readable identifier

Get the contract's address and store it in a variable:
```bash
contractAddress=$(babylond query wasm list-contract-by-code $codeID \
--node=$nodeUrl \
-o json \
| jq -r '.contracts[0]')
```

### 7. Contract Interaction

#### Save Data
To store data in the contract using `tx wasm execute` command:
```bash
# Your data to store
data="This is example plain-text data"

# Convert to hex format (required by contract)
hexData=$(echo -n "$data" | xxd -ps -c0)

# Create the execution message for the save_data function
executeMsg="{\"save_data\":{\"data\":\"$hexData\"}}"

# Send transaction via `tx wasm execute` command
babylond tx wasm execute $contractAddress "$executeMsg" \
--from=$key \
--gas=auto \
--gas-prices=1$feeToken \
--gas-adjustment=1.3 \
--chain-id="$chainId" \
-b=sync \
--yes \
--keyring-backend=test \
--log_format=json \
--home=$homeDir \
--node=$nodeUrl
```

#### Query Data
To retrieve stored data using `query wasm contract` command:
```bash
# Create hash of original data for verification
hashedData=$(echo -n "$data" | sha256sum | cut -f1 -d' ')

# Prepare query message
quer
```

As a reault you should see the following output with data store and finalized status and btc_timestamp information: 

```bash
{
  "data": {
    "finalized": false,
    "latest_finalized_epoch": "644",
    "data": {
      "data": "54686973206973206578616d706c6520706c61696e2d746578742064617461",
      "btc_height": "234328",
      "btc_timestamp": "1738908922",
      "saved_at_btc_epoch": "658"
    }
  }
}
```


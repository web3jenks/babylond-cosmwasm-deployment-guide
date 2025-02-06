# Babylon Smart Contract Deployment Guide

## Prerequisites

In order to deploy the smart contract, you need to have the following installed on your machine:
- [Rust](https://www.rust-lang.org/tools/install)
- [Docker](https://docs.docker.com/get-docker/)

## Deployment

### Step 1: Clone the repository

### Step 2: Update submodules
This project included two git submodules and points to the Babylon and Babylon-contracts repositories.

```bash
$ git submodule update --init --recursive
```
Complete this before preoceeding to the next step.

### Step 3: Build and install babylond cli tool

Ensure you have rust installed: 

```bash
$ rustc --version
rustc 1.81.0 (2dbb1af80 2024-08-20)
```

Build and install babylond cli tool

```bash
$ cd babylon
$ make install
```

Verify the installation by running:

```bash
$ babylond version
main-112821536b0ada40aa29e34b53206f56c61bf631
```

### Step 4: Create a new wallet

Using the `keys add` command, we can create a new wallet and store it in the system keyring. The `--keyring-backend=test` flag is used to specify that we would like to use the test keyring that is stored in the babylond home directory. 

```bash
$ babylond keys add test-key --keyring-backend=test
- address: <your-wallet-address>
  name: test-key
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"<your-wallet-address-public-key>"}'
  type: local

...
```
Where `test-key` is the name of the wallet that will be created.

Store your  mnemonic phrase in case you need to import it again via `--restore` method. 

Verify by using the `keys list` command:

```bash
$ babylond keys list
- name: test-key
  type: local
  address: <your-wallet-address>
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"<your-wallet-address-public-key>"}'
```

Fund your wallet using [L2Sacn faucet](https://babylon-testnet.l2scan.co/faucet). Simply create account and claim tbbn.

Or use following command to get 10 bbn:

```bash
curl https://faucet.testnet.babylonlabs.io/claim -H "Content-Type: multipart/form-data" -d '{"address":"<your-wallet-address>"}'

{"coin":{"denom":"ubbn","amount":"10000000"},"transactionHash":"1C3ED4A9392F9445C7408A2DD3652E04E37F48B0F864FE95E646FF1BC3DBA086","toAddresses":["<your-wallet-address>"]}
```

Check if your balance is updates: 

```bash
babylond q bank balances <your-wallet-address> --node=$nodeUrl

balances:
- amount: "10000000"
  denom: ubbn
pagination:
  total: "1"
```

This endpoint is rate limited to 1 request per day per wallet. 

### Step 5: Load the environment variables into your shell

For Babylon Phase 2 testnet (Babylon chain), please use the `env-phase2-testnet.sh` file. For Phase 3 devnet (multi-staking), please use the `env-phase3-devnet.sh` file.

```bash
$ source env-phase2-testnet.sh
```

check if the environment variables are loaded correctly:

```bash
$ echo $homeDir, $chainId, $feeToken, $key, $keyringBackend, $nodeUrl, $apiUrl
/Users/usr/.babylond, euphrates-0.5.0, ubbn, test-key, --keyring-backend=test, https://rpc-euphrates.devnet.babylonlabs.io, https://lcd-euphrates.devnet.babylonlabs.io
```

### Step 6: Build the smart contract

Babylon storage contract is stored in the `storage-contract` directory. It is a demo contract which enables saving arbitrary data into Babylon with bitcoin timestamped, and checking
whether data is Bitcoin finalized.

Ensures your docker is runing: 

```bash
docker run hello-world
```

Yous hould see a message displaying: 

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

Build the contract using cargo run-script optimizer. This will result in a optimized wasm file built in the artifacts directory.

```bash
cd storage-contract
cargo run-script optimize
```

You should see similar output:

```bash
1.73.0-aarch64-unknown-linux-musl (default)
cargo 1.73.0 (9c4383fb5 2023-08-26)
Building project /code ...
    Updating git repository `https://github.com/babylonlabs-io/bindings`
    Finished release [optimized] target(s) in 0.98s
Optimizing artifacts ...
Optimizing storage_contract-aarch64.wasm ...
Post-processing artifacts...
5dd97a8c3876a0467f48d3b6dae541796b59c64296b51fb32ca25cfb06c1bff3  storage_contract-aarch64.wasm
done
Finished, status of exit status: 0
```

Check the artifacts directory:

```bash
$ ls artifacts
checksums.txt storage_contract-aarch64.wasm
```

### Step 7: Store contract data on Babylon chain

Before we deploy the contract, we need to store the optimized wasm file to Babylon chain using `tx wasm store` command: 

```bash
babylond tx wasm store ./artifacts/storage_contract-aarch64.wasm \
--from=$key \
--gas=auto --gas-prices=1$feeToken --gas-adjustment=1.3 \
--chain-id=“$chainId” \
-b=sync --yes $keyringBackend \
--log_format=json \
--home=$homeDir \
--node=$nodeUrl
```

You should see similar output:

```bash
gas estimate: 1687290
code: 4
codespace: sdk
data: ""
events: []
gas_used: "0"
gas_wanted: "0"
height: "0"
info: ""
logs: []
raw_log: 'signature verification failed; please verify account number (187667) and
  chain-id (bbn-test-5): (unable to verify single signer signature): unauthorized'
timestamp: ""
tx: null
txhash: 2B00B5DF4DAD1E22BF8D1A6098E68830A767D5BD491765E1BB5FFD551B905CA0
```

The txhash is the hash of the transaction. You can use it to query the transaction status:

```bash
babylond q tx 2B00B5DF4DAD1E22BF8D1A6098E68830A767D5BD491765E1BB5FFD551B905CA0 --node=$nodeUrl
```

Or visit testnet chain explorer to check the transaction status [L2Scan Transaction Explorer](https://babylon-testnet.l2scan.co/txs/).

Once the transaction is confirmed, a codeID will be returned to identity the loaded contract. 

```bash
babylond q wasm list-code \
--node $nodeUrl \
-o json | \
jq --arg ADDR "<your-wallet-address>" '.code_infos[] | select(.code_id == "1") | .code_id'
```

### Step 8: Initialize the contract











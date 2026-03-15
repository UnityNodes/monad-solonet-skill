# Monad Solonet — RPC Reference (Verified on Hetzner CX32, Ubuntu 24.04)

## Connection Details

| Parameter | Value |
|-----------|-------|
| **HTTP RPC** | `http://localhost:8080` |
| **Chain ID** | `20143` (hex: `0x4eaf`) |
| **Network name** | `solonet` |
| **Network type** | `devnet` |
| **Dashboard** | `http://localhost:8081` |

**WARNING:** The RPC port is **8080**, NOT the standard EVM 8545!

## Verified RPC Responses (real data from test run)

| Method | Response |
|--------|----------|
| `eth_blockNumber` | `{"result":"0x13bb"}` (block 5051 after ~40min) |
| `eth_chainId` | `{"result":"0x4eaf"}` (= 20143) |
| `eth_getBalance` (Account #0) | `{"result":"0x4b3b4ca85a86c47a098a224000000000"}` (~100M MON) |
| `eth_gasPrice` | `{"result":"0x17bfac7c00"}` (= 102 Gwei) |
| `web3_clientVersion` | `{"result":"Monad/0.13.0"}` |
| `net_peerCount` | **NOT SUPPORTED** — returns `{"error":{"code":-32601,"message":"Method not found"}}` |

### Smart Contract Deployment (verified)

Deployed Counter.sol via `forge create --broadcast`:
- Contract address: `0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512`
- `cast call <addr> "number()"` → `0x0000...0000` (0)
- `cast send <addr> "increment()"` → gasUsed: 49863, status: 1 (success)
- `cast call <addr> "number()"` → `0x0000...0001` (1)
- `cast send <addr> "setNumber(uint256)" 42` → gasUsed: 32943, status: 1 (success)
- `cast call <addr> "number()"` → `0x0000...002a` (42)

## Pre-funded Accounts

Seed: `test test test test test test test test test test test junk`

| # | Address | Private Key |
|---|---------|-------------|
| 0 | `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` | `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80` |
| 1 | `0x70997970C51812dc3A010C7d01b50e0d17dc79C8` | `0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d` |
| 2 | `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC` | `0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a` |

These are standard Hardhat/Foundry test accounts. NEVER use on mainnet.

## Standard EVM JSON-RPC Methods

### Using curl (from host)

```bash
RPC="http://localhost:8080"

# Block number
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

# Chain ID (returns 0x4eaf = 20143)
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'

# Get balance of Account #0
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266","latest"],"id":1}'

# Transaction count (nonce)
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getTransactionCount","params":["0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266","latest"],"id":1}'

# Gas price
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_gasPrice","params":[],"id":1}'

# Get block by number
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["latest",true],"id":1}'

# Client version
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"web3_clientVersion","params":[],"id":1}'

# Estimate gas
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_estimateGas","params":[{"from":"0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266","to":"0x70997970C51812dc3A010C7d01b50e0d17dc79C8","value":"0xDE0B6B3A7640000"}],"id":1}'

# Get transaction receipt
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getTransactionReceipt","params":["0xTX_HASH"],"id":1}'

# Get logs
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getLogs","params":[{"fromBlock":"0x0","toBlock":"latest"}],"id":1}'
```

### Using cast (Foundry — pre-installed in container)

**IMPORTANT:** Foundry is at `/root/.foundry/bin/`, NOT in default PATH.

```bash
# From host (if you have Foundry installed locally)
cast block-number --rpc-url http://localhost:8080
cast chain-id --rpc-url http://localhost:8080
cast balance 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 --rpc-url http://localhost:8080

# From inside container — MUST set PATH first!
docker exec solonet bash -lc 'PATH=/root/.foundry/bin:$PATH cast block-number'
docker exec solonet bash -lc 'PATH=/root/.foundry/bin:$PATH cast chain-id'
docker exec solonet bash -lc 'PATH=/root/.foundry/bin:$PATH cast balance 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266'

# Or enter container interactively
docker exec -it solonet bash
export PATH=/root/.foundry/bin:$PATH
cast block-number          # ETH_RPC_URL=http://localhost:8080 is already set
cast chain-id              # returns 20143

# Deploy contract from inside container (verified working)
cd /tmp && forge init --no-git my-project && cd my-project
forge create --broadcast --rpc-url http://localhost:8080 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  src/Counter.sol:Counter
# NOTE: --broadcast is REQUIRED, without it forge only simulates

# Send transaction
cast send 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 \
  --value 1ether \
  --rpc-url http://localhost:8080 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

## Using with Development Frameworks

### Hardhat

```javascript
// hardhat.config.js
module.exports = {
  solidity: "0.8.24",
  networks: {
    monadSolonet: {
      url: "http://localhost:8080",
      chainId: 20143,
      accounts: [
        "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80",
        "0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d",
        "0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a",
      ],
    },
  },
};
```

### ethers.js v6

```javascript
import { ethers } from "ethers";

const provider = new ethers.JsonRpcProvider("http://localhost:8080");
const wallet = new ethers.Wallet(
  "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80",
  provider
);

const blockNumber = await provider.getBlockNumber();
const balance = await provider.getBalance(wallet.address);
const chainId = (await provider.getNetwork()).chainId; // 20143n
```

### viem

```typescript
import { createPublicClient, createWalletClient, http, defineChain } from "viem";
import { privateKeyToAccount } from "viem/accounts";

const monadSolonet = defineChain({
  id: 20143,
  name: "Monad Solonet",
  nativeCurrency: { name: "Monad", symbol: "MON", decimals: 18 },
  rpcUrls: { default: { http: ["http://localhost:8080"] } },
});

const account = privateKeyToAccount("0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80");

const publicClient = createPublicClient({
  chain: monadSolonet,
  transport: http(),
});

const walletClient = createWalletClient({
  chain: monadSolonet,
  transport: http(),
  account,
});
```

### web3.py

```python
from web3 import Web3

w3 = Web3(Web3.HTTPProvider("http://localhost:8080"))
print(f"Connected: {w3.is_connected()}")
print(f"Block: {w3.eth.block_number}")
print(f"Chain ID: {w3.eth.chain_id}")  # 20143
print(f"Balance: {w3.eth.get_balance('0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266')}")
```

## Staking CLI (pre-installed in container)

```bash
docker exec solonet staking-cli query epoch
docker exec solonet staking-cli query validator-set --type execution
docker exec solonet staking-cli query validator --validator-id 1
```

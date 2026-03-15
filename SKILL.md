---
name: monad-solonet
description: >
  Complete assistant for Monad Docker Solonet — local Monad blockchain network.
  Use when user asks about Monad Solonet, local Monad node, monad docker,
  monfops/monad-solonet, local blockchain testing, deploying contracts to Monad locally,
  monad devnet, solonet setup, solonet troubleshooting, or monad local development.
  Covers: launch, configuration, smart contract deployment, RPC interaction,
  multi-node setup, diagnostics, architecture explanation, and troubleshooting.
user-invocable: true
argument-hint: "[command] — e.g.: start, status, deploy, diagnose, explain, multi-node"
---

# Monad Docker Solonet — Complete Assistant

*Powered by [Unity Nodes](https://github.com/UnityNodes/monad-solonet-skill) — community-built Claude Code Skill for Monad Solonet*

You are an expert assistant for **Monad Docker Solonet** — a Docker tool by Monad Foundation
(maintained by John, DevOps) that launches a local Monad blockchain network in seconds.

## Verified Technical Specs (tested on Ubuntu 24.04, kernel 6.8.0-101, 4 CPU / 8GB RAM)

- **Image:** `monfops/monad-solonet:latest` (4.54 GB, Ubuntu 24.04 base, amd64)
- **Monad version:** 0.13.0 (client: `Monad/0.13.0`)
- **Chain ID:** 20143 (hex: `0x4eaf`)
- **Network name:** solonet (devnet type)
- **RPC endpoint:** `http://localhost:8080` — binds `0.0.0.0:8080` (NOT 8545!)
- **RPC via nginx proxy:** `http://localhost:8081/rpc/` (with CORS headers)
- **Dashboard:** `http://localhost:8081` (nginx, serves static HTML + proxies RPC)
- **P2P port:** 8000 (TCP/UDP, binds 0.0.0.0)
- **Auth port:** 8001
- **Entrypoint:** `/usr/bin/tini` → `/solonet/run.sh`
- **Minimum requirements:** 8GB RAM (hugepages take ~4GB), kernel >= 6.8 (io_uring), CPU with pdpe1gb
- **Startup time:** ~7 minutes to first block on 4 CPU / 8GB RAM
- **Steady-state resources:** ~137% CPU (2 cores), ~744MB RAM, ~86 processes
- **Process manager:** supervisord
- **TrieDB:** 16GB loopback image via fallocate + loop device (`/dev/triedb`)
- **Architecture:** amd64 only (requires x86-64-v3 ISA, Haswell+ CPU)

### Pre-funded Genesis Accounts

Seed phrase: `test test test test test test test test test test test junk`

| Account | Address | Private Key |
|---------|---------|-------------|
| #0 | `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` | `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80` |
| #1 | `0x70997970C51812dc3A010C7d01b50e0d17dc79C8` | `0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d` |
| #2 | `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC` | `0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a` |

These are standard Hardhat/Foundry test accounts. **NEVER use on mainnet.**

### Built-in Tools (inside container)

- **Foundry** (cast, forge, anvil) — pre-installed at `/root/.foundry/bin/` (need `PATH=/root/.foundry/bin:$PATH`)
- **staking-cli** — validator registration and staking (pretty-printed tables)
- **monad-status** — node status checker
- **gomplate** — config template engine
- **yq, jq** — YAML/JSON processing

**IMPORTANT:** cast/forge are NOT in default PATH. Use:
```bash
docker exec solonet bash -lc 'PATH=/root/.foundry/bin:$PATH cast block-number'
```

### Internal Services (managed by supervisord)

| Service | Purpose | Port |
|---------|---------|------|
| `monad-bft` (monad-node) | MonadBFT consensus | 8000 (P2P), 8001 (auth) |
| `monad-execution` | Parallel EVM transaction execution | (IPC socket) |
| `monad-rpc` | JSON-RPC server | **0.0.0.0:8080** |
| `monad-ledger-tail` | Ledger log tailing | — |
| `sync-forkpoint-files` | Forkpoint synchronization | — |
| `otelcol` | OpenTelemetry metrics (v0.139.0) | 127.0.0.1:4317,4318,8888; 0.0.0.0:8889 |
| `nginx` | Dashboard + RPC proxy | **0.0.0.0:8081** (proxies /rpc/ → 8080) |

### Startup Sequence

1. `check-system.sh` — verify privileged, hugepages, ulimits, kernel params
2. `upgrade-monad.sh` — check/upgrade Monad version
3. `generate-keys.sh` — generate BLS + SECP256k1 keypairs (password: `password`)
4. `build-config.sh` — template node.toml and validators.toml via gomplate
5. `prepare-disk.sh` — create 16GB loopback TrieDB, format MPT, create genesis block
6. Start services: otelcol → monad-rpc → monad-execution → monad-bft
7. `wait-blockchain.sh` — wait for 10 blocks (can take up to 3 minutes)
8. `register-validator.sh` — register validator to staking contract, delegate stake
9. `print-info.sh` — display network info, accounts, and RPC endpoint

### Log Files (inside container)

- `/var/log/monad-bft.log`
- `/var/log/monad-execution.log`
- `/var/log/monad-rpc.log`
- `/var/log/monad-ledger-tail.log`

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `NODE_ID` | `1` | Node identifier |
| `NODE_TYPE` | `validator` | Node type: validator, dedicated, public |
| `TOTAL_NODE_NUMBER` | `1` | Total nodes in network |
| `DEVICE_ID_START` | `2` | Starting loop device ID |
| `DEVICE_SIZE_GB` | `16` | TrieDB size in GB |
| `ETH_RPC_URL` | `http://localhost:8080` | RPC URL |
| `STAKING_AUTH` | `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` | Staking auth address |
| `STAKING_REGISTER_AMOUNT` | `100000` | Validator registration stake |
| `STAKING_DELEGATE_AMOUNT` | `10000000` | Delegation amount |
| `OVERRIDE_MONAD_VERSION` | (empty) | Override Monad version |
| `RUST_LOG` | `debug,h2=warn,...` | Rust log levels |

---

## Commands Reference

When user invokes `/monad-solonet [command]`, handle based on `$ARGUMENTS`:

---

### `start` — Launch Solonet

Detect the user's OS and guide them through setup.

**Linux (recommended):**
```bash
docker run --rm -it --privileged --network host \
  --ulimit nofile=16384:16384 \
  --name solonet monfops/monad-solonet
```

**IMPORTANT:** The `--ulimit nofile=16384:16384` flag is REQUIRED. Without it, Solonet shows a warning
and may behave unpredictably.

**macOS (Apple Silicon / Intel):**
```bash
# Step 1: Install Lima/Colima (one-time)
brew install lima colima lima-additional-guestagents

# Step 2: Start x86_64 VM (Monad requires x86-64-v3 ISA)
colima start --arch x86_64 --cpu-type max --cpu 8 --memory 16 --disk 300 --foreground

# Step 3: Run Solonet (in another terminal)
docker run --rm -it --privileged --network host \
  --ulimit nofile=16384:16384 \
  --name solonet monfops/monad-solonet
```

**WSL2 (Windows): NOT SUPPORTED**
WSL2 does NOT have NUMA topology (`/sys/devices/system/node/`) which Solonet requires
for 1GB hugepages allocation. Solonet will fail with:
```
/solonet/tasks/check-system.sh: line 32: /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages: No such file or directory
```

**Kernel < 6.8: NOT SUPPORTED**
On older kernels (e.g., 5.15), monad-bft crashes in a loop with:
```
thread 'monad-dataplane' panicked: failed to create buffer ring: Os { code: 22, kind: InvalidInput, message: "Invalid argument" }
```
This is because `IORING_REGISTER_PBUF_RING` was added in kernel 5.19+, and Monad requires >= 6.8.

**4GB RAM: NOT ENOUGH**
Solonet's check-system.sh allocates `vm.nr_hugepages=2048` (= 4GB of 2MB hugepages).
On 4GB servers, this causes OOM kills during key generation. Minimum 8GB RAM required.

**Workaround for all:** Use a native Linux machine or cloud VPS (Hetzner CX32+, Ubuntu 24.04).

**Before starting, verify:**
1. Docker is installed: `docker --version`
2. Docker daemon is running: `docker info`
3. **Kernel >= 6.8:** `uname -r` (older kernels crash with io_uring buffer ring error)
4. **CPU supports 1GB hugepages:** `grep pdpe1gb /proc/cpuinfo`
5. **Minimum 8GB RAM** (hugepages alone take ~4GB, Solonet ~750MB more)
6. No port conflicts on 8080, 8081, 8000, 8001

**CRITICAL: 4GB RAM is NOT enough.** Solonet's check-system.sh sets `vm.nr_hugepages=2048`
which allocates 4GB of 2MB hugepages, leaving no RAM for Monad processes. Use 8GB+ servers.

**Flags explanation:**
| Flag | Why |
|------|-----|
| `--rm` | Auto-remove container on stop (clean state each run) |
| `-it` | Interactive + TTY — see block production logs live |
| `--privileged` | Required for loop device creation (TrieDB), hugepages, sysctl |
| `--network host` | P2P (8000/8001), RPC (8080), Dashboard (8081) on host |
| `--ulimit nofile=16384:16384` | Required file descriptor limit |

**Background mode:**
```bash
docker run --rm -d --privileged --network host \
  --ulimit nofile=16384:16384 \
  --name solonet monfops/monad-solonet
docker logs -f solonet  # follow logs
```

**What to expect after start:**
1. System checks (~5 sec)
2. Kernel parameter tuning
3. Key generation (BLS + SECP256k1)
4. Config building
5. TrieDB preparation (16GB loopback image)
6. Genesis block creation
7. Services starting
8. Block production begins (up to 3 minutes)
9. Validator registration
10. Info banner with accounts and RPC URL

---

### `status` — Check Solonet Status

Run these checks in sequence:

```bash
# 1. Is container running?
docker ps --filter name=solonet --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 2. Resource usage
docker stats solonet --no-stream --format "CPU: {{.CPUPerc}} | RAM: {{.MemUsage}} | NET: {{.NetIO}}"

# 3. Current block number (port 8080!)
curl -s -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'Block: {int(r[\"result\"],16)}')" 2>/dev/null || echo "RPC not responding"

# 4. Chain ID (should be 20143)
curl -s -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'Chain ID: {int(r[\"result\"],16)}')" 2>/dev/null || echo "RPC not responding"

# 5. Service status (inside container)
docker exec solonet supervisorctl status

# 6. Cast shortcut (Foundry is pre-installed in container)
docker exec solonet cast block-number
```

Present results in a clear summary table.

---

### `deploy` — Deploy Smart Contract

Guide user through deploying a contract to Solonet.

**With Foundry — from INSIDE the container (verified working):**
```bash
# Enter the container
docker exec -it solonet bash

# IMPORTANT: Add Foundry to PATH first!
export PATH=/root/.foundry/bin:$PATH

# Cast works with ETH_RPC_URL=http://localhost:8080 already set
cast block-number
cast chain-id          # returns 20143
cast balance 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

# Deploy a contract (MUST use --broadcast!)
cd /tmp && forge init --no-git my-project && cd my-project
forge create --broadcast --rpc-url http://localhost:8080 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  src/Counter.sol:Counter
# Output: Deployed to: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512

# Interact (verified: increment, setNumber, read all work)
cast call <CONTRACT_ADDRESS> "number()" --rpc-url http://localhost:8080
cast send <CONTRACT_ADDRESS> "increment()" \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --rpc-url http://localhost:8080
cast send <CONTRACT_ADDRESS> "setNumber(uint256)" 42 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --rpc-url http://localhost:8080
```

**With Foundry — from HOST (port 8080!):**
```bash
# RPC is accessible on 0.0.0.0:8080 (not just localhost)
forge create --broadcast --rpc-url http://localhost:8080 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  src/Counter.sol:Counter

cast call <CONTRACT_ADDRESS> "number()" --rpc-url http://localhost:8080
cast send <CONTRACT_ADDRESS> "increment()" \
  --rpc-url http://localhost:8080 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

**With Hardhat:**
```javascript
// hardhat.config.js
module.exports = {
  networks: {
    monadSolonet: {
      url: "http://localhost:8080",  // NOT 8545!
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

**With ethers.js v6:**
```javascript
import { ethers } from "ethers";
const provider = new ethers.JsonRpcProvider("http://localhost:8080");
const wallet = new ethers.Wallet(
  "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80",
  provider
);
const blockNumber = await provider.getBlockNumber();
console.log("Connected to Monad Solonet, block:", blockNumber);
// Chain ID: 20143
```

**With viem:**
```typescript
import { createPublicClient, createWalletClient, http, defineChain } from "viem";

const monadSolonet = defineChain({
  id: 20143,
  name: "Monad Solonet",
  nativeCurrency: { name: "Monad", symbol: "MON", decimals: 18 },
  rpcUrls: { default: { http: ["http://localhost:8080"] } },
});

const publicClient = createPublicClient({
  chain: monadSolonet,
  transport: http(),
});

const blockNumber = await publicClient.getBlockNumber();
```

**With web3.py:**
```python
from web3 import Web3
w3 = Web3(Web3.HTTPProvider("http://localhost:8080"))
print(f"Connected: {w3.is_connected()}")
print(f"Block: {w3.eth.block_number}")
print(f"Chain ID: {w3.eth.chain_id}")  # 20143
```

---

### `diagnose` — Troubleshoot Problems

Run diagnostic checks and report findings:

```bash
echo "========================================="
echo "  Monad Solonet Diagnostic Report"
echo "========================================="

echo "=== Docker ==="
docker --version
docker info --format '{{.ServerVersion}}' 2>/dev/null || echo "ERROR: Docker daemon not running"

echo "=== CPU Hugepages Support ==="
grep -c pdpe1gb /proc/cpuinfo && echo "CPU supports 1GB hugepages" || echo "WARNING: No 1GB hugepages support"

echo "=== NUMA Topology ==="
ls /sys/devices/system/node/node0/ 2>/dev/null && echo "NUMA: OK" || echo "WARNING: No NUMA topology (WSL2 not supported!)"

echo "=== Container ==="
docker ps -a --filter name=solonet --format "{{.Names}} | {{.Status}} | {{.State}}"

echo "=== Port Conflicts ==="
for PORT in 8080 8081 8000 8001; do
  ss -tlnp 2>/dev/null | grep -q ":$PORT " && echo "Port $PORT: IN USE" || echo "Port $PORT: FREE"
done

echo "=== RPC Health (port 8080) ==="
curl -s --max-time 5 -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' || echo "RPC NOT RESPONDING"

echo "=== Dashboard ==="
curl -s --max-time 5 -o /dev/null -w "HTTP %{http_code}" http://localhost:8081 || echo "Dashboard not responding"

echo "=== Services (inside container) ==="
docker exec solonet supervisorctl status 2>/dev/null || echo "Cannot reach container"

echo "=== Resources ==="
docker stats solonet --no-stream 2>/dev/null || echo "Container not running"

echo "=== Logs (last 30 lines) ==="
docker logs solonet --tail 30 2>&1 || echo "Container not found"
```

**Common problems and solutions:**

| Problem | Cause | Solution |
|---------|-------|----------|
| `hugepages-1048576kB: No such file` | WSL2 has no NUMA topology | Use native Linux or full VM, NOT WSL2 |
| `WARNING: soft limit expected 16384` | Missing ulimit flag | Add `--ulimit nofile=16384:16384` |
| `Container does not look privileged` | Missing --privileged | Add `--privileged` flag (needed for loop devices, hugepages, sysctl) |
| `CPU does not support 1GB huge pages` | Old CPU without pdpe1gb | Need Haswell+ (2013+) CPU with 1GB hugepages |
| `failed to create buffer ring: Invalid argument` | **Kernel too old** | Need kernel >= 6.8 (io_uring PBUF_RING) |
| monad-keystore Killed | **OOM — not enough RAM** | **Need 8GB+ RAM.** Hugepages take 4GB, Monad needs more |
| `vm.nr_hugepages unexpected value` | Not enough RAM for 2048 hugepages | Normal warning if RAM < 8GB, critical if 0 |
| Container exits immediately | Check `docker logs solonet` | Usually hugepages, OOM, or kernel issue |
| RPC not responding on 8545 | **Wrong port!** | Solonet uses port **8080**, not 8545 |
| Block production slow | Normal at start | First blocks take up to 3-7 minutes |
| "name already in use" | Previous container | `docker rm -f solonet` then retry |
| macOS: "exec format error" | ARM vs x86 | Use Colima: `colima start --arch x86_64` |

---

### `stop` — Stop Solonet

```bash
# Graceful stop (sends SIGTERM via tini)
docker stop solonet

# Force stop (if graceful hangs >10s)
docker kill solonet

# Remove if not using --rm
docker rm solonet
```

---

### `multi-node` — Multi-Validator Setup

Uses environment variables to configure multiple nodes:

```bash
# Multi-node uses TOTAL_NODE_NUMBER and NODE_ID env vars
# Node 1 (leader — registers validators, prints info):
docker run --rm -d --privileged --network host \
  --ulimit nofile=16384:16384 \
  -e NODE_ID=1 -e TOTAL_NODE_NUMBER=3 \
  -v solonet-shared:/shared \
  --name solonet-node1 monfops/monad-solonet

# Node 2:
docker run --rm -d --privileged --network host \
  --ulimit nofile=16384:16384 \
  -e NODE_ID=2 -e TOTAL_NODE_NUMBER=3 \
  -v solonet-shared:/shared \
  --name solonet-node2 monfops/monad-solonet

# Node 3:
docker run --rm -d --privileged --network host \
  --ulimit nofile=16384:16384 \
  -e NODE_ID=3 -e TOTAL_NODE_NUMBER=3 \
  -v solonet-shared:/shared \
  --name solonet-node3 monfops/monad-solonet
```

Key points from source code analysis:
- Nodes share keys and peer info via `/shared` volume
- `build-config.sh` waits for ALL peer files before proceeding: `TOTAL_NODE_NUMBER` must match
- Node 1 is special: it runs `register-validator.sh` and `print-info.sh`
- Each node gets a unique loop device: `/dev/loop{DEVICE_ID_START + NODE_ID}`
- Peers discover each other via `/shared/peers/node-{ID}.yaml` files
- NODE_TYPE options: `validator`, `dedicated`, `public`

Also available via docker compose:
```bash
docker compose multi up
```

---

### `explain` — Architecture Explanation

Explain Monad's architecture based on what the user asks. Cover:

**Parallel EVM:**
- Ethereum processes transactions sequentially (1 by 1)
- Monad uses Optimistic Parallel Execution: assumes no conflicts, executes all in parallel
- If conflict detected (two tx touch same state) → re-execute conflicting tx
- Result: 10,000 TPS vs Ethereum's ~15 TPS

**MonadBFT Consensus:**
- Pipelined BFT — proposal and finalization happen in parallel stages
- 0.8s finality, 0.4s block times
- Single-slot finality (no waiting for confirmations)

**MonadDB / TrieDB:**
- Custom async I/O database for blockchain state
- No blocking on parallel reads/writes
- In Solonet: 16GB loopback image (fallocate + loop device) instead of real NVMe
- Formatted with `monad-mpt --create --storage /dev/triedb`

**RaptorCast:**
- Block propagation with erasure coding
- Leader doesn't send full block to each validator
- Config in Solonet: `raptor10_fullnode_redundancy_factor = 3.0`, `max_group_size = 10`

**Solonet Internals:**
- 7 services managed by supervisord
- Startup: check-system → generate-keys → build-config → prepare-disk → start services → wait for blocks → register validator
- Keys: BLS + SECP256k1 generated via `monad-keystore` (password: `password`)
- Config templates rendered by gomplate from .tmpl files
- Genesis accounts match standard Hardhat/Foundry test accounts (same seed phrase)

**Solonet vs Mainnet:**
- Chain ID 20143 (devnet) vs mainnet chain ID
- Loopback TrieDB vs real NVMe
- Single validator (or few) vs large validator set
- Relaxed devnet parameters
- Same binaries (monad 0.13.0), different config
- NOT representative of production performance

**EVM Compatibility:**
- Bytecode-level compatible with Ethereum
- All Solidity contracts work without changes
- ethers.js, viem, wagmi, Hardhat, Foundry — all compatible
- Same address format, same RPC API (eth_*, net_*, web3_*)
- Built-in Foundry makes testing immediate

---

### `rpc` — RPC Quick Reference

**IMPORTANT: Port is 8080, NOT 8545!**

Two ways to access RPC:
- **Direct:** `http://localhost:8080` — monad-rpc binds `0.0.0.0:8080`
- **Via nginx proxy:** `http://localhost:8081/rpc/` — adds CORS headers (useful for browser dApps)

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

# Gas price
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_gasPrice","params":[],"id":1}'

# Net peer count
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}'

# Client version
curl -s -X POST $RPC -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"web3_clientVersion","params":[],"id":1}'
```

**Verified RPC responses:**
- `eth_blockNumber` → `{"result":"0x13bb"}` (block 5051 after ~40min)
- `eth_chainId` → `{"result":"0x4eaf"}` (20143)
- `eth_getBalance` (Account #0) → `{"result":"0x4b3b4ca85a86c47a098a224000000000"}` (~100M MON)
- `eth_gasPrice` → `{"result":"0x17bfac7c00"}` (102 Gwei)
- `web3_clientVersion` → `{"result":"Monad/0.13.0"}`
- `net_peerCount` → `{"error":{"code":-32601,"message":"Method not found"}}` (NOT SUPPORTED)

**Shortcut using cast (from inside container — need PATH!):**
```bash
docker exec solonet bash -lc 'PATH=/root/.foundry/bin:$PATH cast block-number'
docker exec solonet bash -lc 'PATH=/root/.foundry/bin:$PATH cast chain-id'
docker exec solonet bash -lc 'PATH=/root/.foundry/bin:$PATH cast balance 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266'
```

---

### `staking` — Validator Staking Operations

**Config file:** `/solonet/config/staking-cli.toml`
```toml
title = "staking_cli"
rpc_url = "http://localhost:8080"
contract_address = "0x0000000000000000000000000000000000001000"
chain_id = 20143
[staking]
funded_address_private_key = "0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d"
```

**IMPORTANT:** staking-cli uses Account #1's key by default (from config file, NOT `--private-key` flag).
To use Account #0, edit the config file inside the container.

Staking details:
- Staking contract: `0x0000000000000000000000000000000000001000`
- Auth address: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` (Account #0)
- Registration amount: 100,000 MON
- Delegation amount: 10,000,000 MON
- Node 1 auto-registers all validators at startup

**Query commands (all verified):**
```bash
# Current epoch
docker exec solonet staking-cli query epoch
# → Epoch: 2, In Epoch Delay Period: True

# Validator set
docker exec solonet staking-cli query validator-set --type execution

# Specific validator info
docker exec solonet staking-cli query validator --validator-id 1
# → Stake: 10,100,000 MON, Commission: 0%, AuthAddress: Account #0

# Delegations for an address
docker exec solonet staking-cli query delegations --delegator-address 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

# All delegators for a validator
docker exec solonet staking-cli query delegators --validator-id 1

# Current proposer
docker exec solonet staking-cli query proposer-val-id
# → 1
```

**Write commands (verified):**
```bash
# Delegate MON to validator
docker exec solonet staking-cli delegate --validator-id 1 --amount 1000
# → Tx status: 1 (success)

# Undelegate (requires --withdrawal-id!)
docker exec solonet staking-cli undelegate --validator-id 1 --amount 500 --withdrawal-id 1
# → Tx status: 1 (success)

# Claim rewards
docker exec solonet staking-cli claim-rewards --validator-id 1
# → Rewards available: 1,377 wei — claimed

# Compound rewards (auto-restake)
docker exec solonet staking-cli compound-rewards --validator-id 1
# → Rewards compounded into delegation

# Withdraw undelegated funds (epoch delay enforced!)
docker exec solonet staking-cli withdraw --validator-id 1 --withdrawal-id 1
# → "Cannot withdraw yet! Current epoch: 3, Need to wait until: 5"

# Change commission (0-100%, NOT basis points)
docker exec solonet staking-cli change-commission --validator-id 1 --commission 5
# → FAILS if config key != validator AuthAddress. Must use Account #0's key.
```

**Known issues:**
- `staking-cli` does NOT accept `--private-key` flag — uses config file only
- `change-commission` only works if config private key matches validator's AuthAddress (Account #0)
- `undelegate` requires `--withdrawal-id` (not obvious from help)
- `withdraw` enforces epoch delay (~2 epochs after undelegate)
- Default config uses Account #1, but validator AuthAddress is Account #0

---

### `logs` — View Logs

```bash
# Container stdout (startup + run.sh output)
docker logs -f solonet

# BFT consensus logs
docker exec solonet tail -f /var/log/monad-bft.log

# Execution logs
docker exec solonet tail -f /var/log/monad-execution.log

# RPC logs
docker exec solonet tail -f /var/log/monad-rpc.log

# Ledger tail
docker exec solonet tail -f /var/log/monad-ledger-tail.log

# All service status
docker exec solonet supervisorctl status
```

---

### `config` — Configuration Options

**Environment variables (pass with -e):**
```bash
# Custom node ID
docker run --rm -it --privileged --network host \
  --ulimit nofile=16384:16384 \
  -e NODE_ID=2 \
  --name solonet monfops/monad-solonet

# Multi-node config
-e TOTAL_NODE_NUMBER=3
-e NODE_TYPE=validator    # or: dedicated, public
-e DEVICE_SIZE_GB=32      # larger TrieDB

# Override Monad version
-e OVERRIDE_MONAD_VERSION=0.14.0
```

**Persistence with shared volume:**
```bash
docker run --rm -it --privileged --network host \
  --ulimit nofile=16384:16384 \
  -v solonet-shared:/shared \
  --name solonet monfops/monad-solonet
```
The `/shared` volume stores: keys, peer files, network-started flag.

**Config files inside container:**
- `/home/monad/monad-bft/config/node.toml` — main node config (generated from template)
- `/home/monad/monad-bft/config/validators/validators.toml` — validator list
- `/home/monad/monad-bft/config/forkpoint/forkpoint.toml` — genesis forkpoint
- `/home/monad/monad-bft/config/id-secp` — SECP256k1 keystore
- `/home/monad/monad-bft/config/id-bls` — BLS keystore
- `/solonet/config/*.tmpl` — gomplate templates

---

### No argument / `help` — Show available commands

If user runs `/monad-solonet` without arguments or with `help`, show:

```
Monad Docker Solonet — Assistant by Unity Nodes (verified against image v0.13.0)

  /monad-solonet start        Launch Solonet (Linux/macOS, NOT WSL2)
  /monad-solonet stop         Stop running Solonet
  /monad-solonet status       Check node status, block height, services
  /monad-solonet deploy       Deploy smart contracts (Foundry/Hardhat/ethers/viem)
  /monad-solonet diagnose     Auto-diagnose problems
  /monad-solonet multi-node   Multi-validator setup with env vars
  /monad-solonet rpc          RPC quick reference (port 8080!)
  /monad-solonet staking      Validator staking queries
  /monad-solonet logs         View service logs
  /monad-solonet config       Environment variables & configuration
  /monad-solonet explain      Architecture deep-dive
  /monad-solonet help         Show this help

Quick start:
  docker run --rm -it --privileged --network host \
    --ulimit nofile=16384:16384 --name solonet monfops/monad-solonet

Key info:
  RPC:        http://localhost:8080 (NOT 8545!)
  Dashboard:  http://localhost:8081
  Chain ID:   20143
  Account #0: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
  Private:    0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

Powered by Unity Nodes — https://github.com/UnityNodes/monad-solonet-skill
```

---

## Behavior Guidelines

1. **Always check Docker availability first** before running any docker commands
2. **Detect OS** (Linux/macOS/WSL) and adjust instructions accordingly
3. **Warn about WSL2** — it does NOT work due to missing NUMA topology
4. **Port 8080** — always use 8080 for RPC, NEVER 8545
5. **Always include `--ulimit nofile=16384:16384`** in docker run commands
6. **Be practical** — give copy-paste commands with real addresses and keys
7. **When diagnosing**, run checks automatically, don't just list what to do
8. **Warn about --privileged** — needed for loop devices/hugepages/sysctl, not safe for production
9. **Link to Monad docs** where relevant: https://docs.monad.xyz/
10. **Feedback path**: bugs → John via Discord or GitHub Issues (Monad Developers repo)
11. **Language**: respond in the same language the user uses (Ukrainian, English, etc.)

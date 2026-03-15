# Monad Solonet — Architecture Reference (Verified on Hetzner CX32, Ubuntu 24.04)

## Image Details

- **Image:** `monfops/monad-solonet:latest`
- **Size:** 4.54 GB
- **Base:** Ubuntu 24.04 (amd64)
- **Built:** 2026-03-12
- **Monad version:** 0.13.0
- **OTEL version:** 0.139.0

## What's Inside the Image (layer analysis)

| Layer | Size | Content |
|-------|------|---------|
| Ubuntu 24.04 base | 78 MB | Base OS |
| System packages | 301 MB | aria2, curl, git, jq, nginx, supervisor, tini, vim, yq, python3 |
| staking-sdk-cli | 182 MB | Python-based staking CLI tool |
| monad-status | 14 KB | Node status script |
| Foundry | 392 MB | cast, forge, anvil (Ethereum dev toolkit) |
| OpenTelemetry | 204 MB | otelcol metrics collector |
| Monad 0.13.0 | 3.27 GB | monad-node, monad-bft, monad-execution, monad-rpc, monad-mpt, monad-keystore |
| gomplate | 108 MB | Config template engine |
| Dashboard | 17 KB | nginx dashboard config + files |
| Solonet scripts | 21 KB | run.sh, tasks/, lib/, config/ |

## Startup Sequence (from run.sh)

```
1. supervisord starts (manages all services)
2. check-system.sh
   ├── Verify --privileged (capabilities check)
   ├── Verify CPU 1GB hugepages (pdpe1gb flag)
   ├── Verify ulimit nofile=16384
   ├── Set kernel params:
   │   ├── 4x 1GB hugepages on node0
   │   ├── 2048x 2MB hugepages
   │   ├── sysctl net.ipv4.tcp_rmem/wmem = 4096 62500000 62500000
   │   └── sysctl net.core.rmem_max/wmem_max = 62500000
   └── Verify all sysctl values

3. upgrade-monad.sh (check/upgrade Monad version)

4. generate-keys.sh
   ├── Create SECP256k1 keystore (password: "password")
   ├── Create BLS keystore (password: "password")
   ├── Sign name record for peer discovery
   └── Write peer YAML to /shared/peers/node-{ID}.yaml

5. build-config.sh
   ├── Wait for ALL peer files (TOTAL_NODE_NUMBER)
   ├── Copy keys to /home/monad/monad-bft/config/
   ├── Render node.toml from template (gomplate)
   └── Render validators.toml from template (gomplate)

6. prepare-disk.sh
   ├── fallocate 16GB image file
   ├── losetup loop device
   ├── monad-mpt --create (format TrieDB)
   ├── Copy genesis forkpoint
   └── monad --chain monad_devnet --nblocks 0 (genesis block)

7. Start services:
   ├── otelcol (metrics)
   ├── monad-rpc (JSON-RPC on port 8080)
   ├── monad-execution (parallel EVM)
   └── monad-bft (consensus)

8. wait-blockchain.sh
   ├── Wait for block > 0
   ├── Wait for 10 blocks produced
   ├── Touch /shared/node-{ID}-started
   └── Wait for all nodes, touch /shared/network-started

9. Start additional services:
   ├── monad-ledger-tail
   └── sync-forkpoint-files

10. register-validator.sh (NODE_ID=1 only)
    ├── Register validator via staking-cli
    ├── Delegate stake (10,000,000)
    └── Print validator set

11. print-info.sh (NODE_ID=1 only)
    └── Print network info, accounts, RPC URL

12. tail -f /dev/null (keep container alive)
```

## Node Configuration (node.toml)

Generated from `/solonet/config/node.toml.tmpl` via gomplate:

```toml
beneficiary = "0x0000000000000000000000000000000000000000"
node_name = "node-1"
network_name = "solonet"
chain_id = 20143
ipc_tx_batch_size = 500
ipc_max_queued_batches = 6
ipc_queued_batches_watermark = 3
statesync_threshold = 600

[network]
bind_address_host = "0.0.0.0"
bind_address_port = 8000
authenticated_bind_address_port = 8001
max_rtt_ms = 300
max_mbps = 1000

[fullnode_raptorcast]
enable_publisher = true
enable_client = true
raptor10_fullnode_redundancy_factor = 3.0
max_group_size = 10
round_span = 240
# ... more RaptorCast settings

[peer_discovery]
self_address = "<container_ip>:8000"
refresh_period = 120
min_num_peers = 0
max_num_peers = 200
```

## Multi-Node Architecture

```
/shared/ volume (shared between all nodes)
├── keys/
│   ├── 1/id-secp, 1/id-bls    (node 1 keys)
│   ├── 2/id-secp, 2/id-bls    (node 2 keys)
│   └── 3/id-secp, 3/id-bls    (node 3 keys)
├── peers/
│   ├── node-1.yaml             (node 1 peer info + keys)
│   ├── node-2.yaml             (node 2 peer info + keys)
│   └── node-3.yaml             (node 3 peer info + keys)
├── node-1-started              (flag file)
├── node-2-started              (flag file)
└── network-started             (set when all nodes ready)
```

Each peer YAML contains:
```yaml
node_name: node-1
node_type: validator
address: <ip>:8000
auth_port: 8001
self_record_seq_num: 0
self_name_record_sig: "<signature>"
secp256k1:
  public_key: "<pubkey>"
  private_key: "<privkey>"
bls:
  public_key: "<pubkey>"
  private_key: "<privkey>"
```

## Monad Architecture (General)

### Optimistic Parallel Execution
- Transactions execute simultaneously, assuming no conflicts
- Post-execution conflict detection
- Conflicting transactions re-executed sequentially
- ~95% of transactions don't conflict → massive throughput gain

### MonadBFT Consensus
- Pipelined BFT (HotStuff family)
- 0.4s block times, 0.8s finality
- Single-slot finality

### MonadDB / TrieDB
- Custom async I/O storage engine
- Merkle Patricia Trie (Ethereum compatible)
- In Solonet: 16GB loopback file via loop device
- In production: dedicated NVMe drive (512-byte LBA)

### RaptorCast
- Erasure-coded block propagation
- Configured in Solonet: redundancy_factor=3.0, max_group_size=10

### EVM Compatibility
- Bytecode-level (Solidity, Vyper)
- Same RPC API (eth_*, net_*, web3_*)
- Same address format, same tx format (EIP-1559)
- Tools: Hardhat, Foundry, Remix, ethers.js, viem, web3.py

## Minimum System Requirements (Verified)

| Requirement | Minimum | Tested |
|-------------|---------|--------|
| **RAM** | 8 GB | 4GB OOM kills during key generation; 8GB works |
| **CPU** | x86-64 with `pdpe1gb` flag | Haswell+ (Intel 4th gen / AMD Zen+) |
| **Kernel** | >= 6.8 | 5.15 crashes (io_uring); 6.8.0-101 works |
| **Disk** | 20 GB free | 16GB TrieDB + 4.54GB image |
| **OS** | Ubuntu 24.04 (recommended) | Also works on other Linux with kernel >= 6.8 |
| **Docker** | 20.10+ | With `--privileged` and `--network host` |

**NOT supported:**
- WSL2 (no NUMA topology at `/sys/devices/system/node/`)
- Kernel < 5.19 (no `IORING_REGISTER_PBUF_RING`)
- ARM64 / Apple Silicon (image is amd64 only, use Colima with `--arch x86_64`)
- 4GB RAM or less (hugepages alone take 4GB)

**Tested performance (Hetzner CX32: 4 vCPU, 8GB RAM, Ubuntu 24.04, kernel 6.8.0-101):**
- Startup to first block: ~7 minutes
- Steady-state CPU: ~137% (2 cores)
- Steady-state RAM: ~744 MiB
- Process count: ~86
- Block time: sub-second after initial warmup

## Resources

- Monad Official: https://www.monad.xyz/
- Documentation: https://docs.monad.xyz/
- Consensus source: https://github.com/category-labs/monad-bft
- Execution source: https://github.com/category-labs/monad
- Staking CLI: https://github.com/monad-developers/staking-sdk-cli

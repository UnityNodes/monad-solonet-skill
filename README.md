# Monad Solonet — Claude Code Skill

*By [Unity Nodes](https://github.com/UnityNodes) — community-built skill for Monad developers and validators*

> A [Claude Code](https://claude.com/claude-code) skill for working with **Monad Docker Solonet** — the local Monad blockchain network by Monad Foundation.
>
> Tested on real hardware. All data verified against `monfops/monad-solonet:latest` (v0.13.0).

## Installation

**One command (recommended):**
```bash
npx skills add UnityNodes/monad-solonet-skill -g -y
```

**Or manually:**
```bash
git clone https://github.com/UnityNodes/monad-solonet-skill.git

# For one project
cp -r monad-solonet-skill YOUR_PROJECT/.claude/skills/

# For all projects (global)
cp -r monad-solonet-skill ~/.claude/skills/
```

Then type `/monad-solonet deploy` in Claude Code.

## What is this?

A Claude Code skill with **12 commands** that turns Claude into a full assistant for Monad Solonet development. Instead of reading docs and debugging setup issues, developers type `/monad-solonet` and get instant, verified help.

Based on testing across 3 Linux servers, deploying smart contracts, and testing the full validator staking lifecycle — **17 problems documented with solutions**.

## Verified Specs

| Parameter | Value |
|-----------|-------|
| Image | `monfops/monad-solonet:latest` (4.54 GB) |
| Monad | v0.13.0 on Ubuntu 24.04 (amd64) |
| RPC | `http://localhost:8080` (**NOT 8545!**) |
| Dashboard | `http://localhost:8081` |
| Chain ID | `20143` |
| P2P / Auth | Port 8000 / 8001 |
| Genesis seed | `test test test test test test test test test test test junk` |
| Account #0 | `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` |
| Built-in tools | Foundry, staking-cli, monad-status, gomplate, yq, jq |

## Commands

| Command | Description |
|---------|-------------|
| `/monad-solonet start` | Launch with OS-specific instructions (Linux, macOS via Colima) |
| `/monad-solonet stop` | Graceful & force stop |
| `/monad-solonet status` | Block height, services, CPU/RAM usage |
| `/monad-solonet deploy` | Deploy contracts via Foundry, Hardhat, ethers.js, viem, web3.py |
| `/monad-solonet diagnose` | Auto-detect and fix problems |
| `/monad-solonet staking` | Verified delegation, rewards, and commission commands |
| `/monad-solonet multi-node` | Multi-validator setup via `docker compose multi up` |
| `/monad-solonet rpc` | Every RPC method with real response examples |
| `/monad-solonet logs` | All service log locations |
| `/monad-solonet config` | Environment variables & config file paths |
| `/monad-solonet explain` | Architecture deep-dive (Parallel EVM, MonadBFT, TrieDB, RaptorCast) |
| `/monad-solonet help` | All commands at a glance |

## Key Findings

> Tested on Hetzner CX32 (4 vCPU, 8GB RAM, Ubuntu 24.04, kernel 6.8.0-101)

1. **RPC port is 8080**, not the standard 8545
2. **`--ulimit nofile=16384:16384` is required** — missing from official Docker Hub command
3. **Kernel >= 6.8 required** — older kernels crash with io_uring buffer ring error
4. **8GB RAM minimum** — hugepages take 4GB; 4GB servers get OOM-killed silently
5. **Foundry is pre-installed** but NOT in PATH — use `PATH=/root/.foundry/bin:$PATH`
6. **staking-cli uses config file**, not `--private-key` flag
7. **Default staking config uses Account #1**, but validator AuthAddress is Account #0
8. **`undelegate` requires `--withdrawal-id`** — not mentioned in help text
9. **`forge create` requires `--broadcast`** — without it, simulation only
10. **`net_peerCount` is NOT supported** — returns "Method not found"
11. **Smart contracts deploy and work** — verified Counter.sol end-to-end
12. **Full staking lifecycle works** — delegate, claim, compound, undelegate, withdraw
13. **7 services** managed by supervisord
14. **16GB TrieDB** created as loopback image via loop device
15. **Startup takes ~7 minutes** to first block
16. **~12,210 blocks/hour** steady-state production
17. **Keys use password `password`** (BLS + SECP256k1)

## Minimum Requirements

| Requirement | Why |
|-------------|-----|
| **8GB RAM** | Hugepages take 4GB, Monad needs ~750MB more |
| **Kernel >= 6.8** | io_uring buffer rings required |
| **CPU with pdpe1gb** | 1GB hugepages (Intel Haswell+ / AMD Zen+) |
| **Linux or macOS** | Linux natively, macOS via Colima `--arch x86_64` |

## Quick Start

```bash
# Linux
docker run --rm -it --privileged --network host \
  --ulimit nofile=16384:16384 \
  --name solonet monfops/monad-solonet

# Wait ~7 minutes for "NETWORK STARTED!" banner, then:
curl -s -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

## File Structure

```
monad-solonet-skill/
├── SKILL.md                  # Main skill (12 commands, verified specs)
├── references/
│   ├── architecture.md       # Image layers, startup sequence, node config
│   ├── troubleshooting.md    # 17 verified problems with solutions
│   └── rpc-reference.md      # RPC methods, cast, Hardhat, ethers, viem, web3.py
└── README.md
```

## How This Was Verified

1. Pulled `monfops/monad-solonet:latest` and analyzed with `docker inspect` + `docker history`
2. Extracted and analyzed all 15 scripts in `/solonet/`
3. Tested on Hetzner Dedicated (kernel 5.15) — confirmed io_uring crash
4. Tested on Hetzner CX23 (4GB RAM) — confirmed OOM kill
5. **Successfully launched on Hetzner CX32** (8GB RAM, kernel 6.8)
6. Deployed Counter.sol, verified all contract interactions
7. Tested full staking lifecycle: delegate, claim-rewards, compound-rewards, undelegate, withdraw
8. Tested all 8 staking-cli query commands
9. Documented 17 problems with solutions

## Contributing

1. Fork this repository
2. Edit `SKILL.md` or files in `references/`
3. Test with Claude Code
4. Submit a PR

## Links

- [Monad Documentation](https://docs.monad.xyz/)
- [Monad Docker Solonet on Docker Hub](https://hub.docker.com/r/monfops/monad-solonet)
- [Foundry Book](https://book.getfoundry.sh/)
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code)

---

*Built and tested by [Unity Nodes](https://github.com/UnityNodes), March 2026*
*Hetzner CX32 (4 vCPU, 8GB RAM, Ubuntu 24.04, kernel 6.8.0-101) · Monad Solonet v0.13.0*

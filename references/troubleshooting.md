# Monad Solonet — Troubleshooting Guide (Verified)

## Quick Diagnostic Script

```bash
#!/bin/bash
echo "========================================="
echo "  Monad Solonet Diagnostic Report"
echo "========================================="

echo "[System]"
echo "OS: $(uname -s -r -m)"
echo "Docker: $(docker --version 2>/dev/null || echo 'NOT INSTALLED')"
echo "Docker daemon: $(docker info --format '{{.ServerVersion}}' 2>/dev/null || echo 'NOT RUNNING')"
echo "CPU hugepages: $(grep -c pdpe1gb /proc/cpuinfo 2>/dev/null && echo 'supported' || echo 'NOT supported')"
echo "NUMA: $(ls /sys/devices/system/node/node0/ &>/dev/null && echo 'OK' || echo 'MISSING (WSL2?)')"

echo "[Container]"
CONTAINER=$(docker ps -a --filter name=solonet --format "{{.Names}}|{{.Status}}|{{.State}}" 2>/dev/null)
if [ -z "$CONTAINER" ]; then
    echo "Status: No solonet container found"
else
    echo "Status: $CONTAINER"
    echo "Resources: $(docker stats solonet --no-stream --format 'CPU={{.CPUPerc}} RAM={{.MemUsage}}' 2>/dev/null || echo 'N/A')"
fi

echo "[Ports]"
for PORT in 8080 8081 8000 8001; do
    ss -tlnp 2>/dev/null | grep -q ":$PORT " && echo "Port $PORT: IN USE" || echo "Port $PORT: FREE"
done

echo "[RPC - port 8080]"
RPC_RESPONSE=$(curl -s --max-time 5 -X POST http://localhost:8080 \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' 2>/dev/null)
if [ -z "$RPC_RESPONSE" ]; then
    echo "RPC: NOT RESPONDING"
else
    echo "RPC: ACTIVE — $RPC_RESPONSE"
fi

echo "[Services]"
docker exec solonet supervisorctl status 2>/dev/null || echo "Cannot connect to container"

echo "[Recent Logs]"
docker logs solonet --tail 20 2>&1 || echo "No logs available"
echo "========================================="
```

## Verified Problems and Solutions

> All problems below were encountered and verified during real testing on:
> - WSL2 (Windows 11, kernel 5.15.x) — NUMA topology failure
> - Hetzner CX23 (4GB RAM, Ubuntu 24.04, kernel 6.8.0-101) — OOM failure
> - Hetzner CX32 (8GB RAM, Ubuntu 24.04, kernel 6.8.0-101) — SUCCESS
> - Hetzner dedicated (Ubuntu 22.04, kernel 5.15) — io_uring failure

### 1. WSL2: hugepages path not found (CONFIRMED BUG)

**Symptom:**
```
/solonet/tasks/check-system.sh: line 32: /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages: No such file or directory
```

**Root cause:** WSL2 kernel (5.15.x-microsoft-standard-WSL2) does not expose NUMA topology.
The path `/sys/devices/system/node/node0/` does not exist. Solonet's `check-system.sh` tries
to write to this path and fails with `set -euo pipefail`.

**WSL2 has:** `/sys/kernel/mm/hugepages/hugepages-1048576kB/` (different path)
**Solonet expects:** `/sys/devices/system/node/node0/hugepages/hugepages-1048576kB/`

**Solution:** WSL2 is NOT supported. Use:
- Native Linux machine
- Full VM (VirtualBox, VMware, Hyper-V with nested NUMA)
- macOS with Colima (`--arch x86_64`)
- Cloud Linux instance (AWS, GCP, etc.)

**Potential fix for Monad team:** check-system.sh could fallback to
`/sys/kernel/mm/hugepages/` when NUMA path is unavailable.

### 2. Missing ulimit flag

**Symptom:**
```
WARNING: soft limit expected 16384, got 1024
WARNING: hard limit expected 16384, got 1048576
⚠️ Please set --ulimit nofile=16384:16384
```

**Solution:** Always include the flag:
```bash
docker run --rm -it --privileged --network host \
  --ulimit nofile=16384:16384 \
  --name solonet monfops/monad-solonet
```

### 3. Container exits immediately (--rm hides logs)

**Symptom:** `docker run --rm -d ...` succeeds but container disappears instantly.

**Root cause:** `--rm` removes container on exit, so `docker logs` can't show why it failed.

**Solution:** Run in foreground to see the error:
```bash
docker run --rm -it --privileged --network host \
  --ulimit nofile=16384:16384 \
  --name solonet monfops/monad-solonet
```
Or run without `--rm` to preserve logs:
```bash
docker run -d --privileged --network host \
  --ulimit nofile=16384:16384 \
  --name solonet monfops/monad-solonet
# After it stops:
docker logs solonet
docker rm solonet
```

### 4. RPC not responding on port 8545

**Symptom:** `curl localhost:8545` → connection refused

**Root cause:** Solonet uses port **8080**, NOT the standard EVM 8545.
The ENV variable is set: `ETH_RPC_URL=http://localhost:8080`

**Solution:** Use port 8080:
```bash
curl -s -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

### 5. CPU does not support 1GB huge pages

**Symptom:** `❌ CPU does not support 1GB huge pages`

**Diagnosis:**
```bash
grep pdpe1gb /proc/cpuinfo
```

**Root cause:** CPU is too old (pre-Haswell, before 2013) or running in a VM
that doesn't expose the pdpe1gb flag.

**Solution:** Use hardware with Haswell+ CPU (Intel 4th gen or newer, AMD Zen+).

### 6. "name already in use"

**Symptom:** `The container name "/solonet" is already in use`

**Solution:**
```bash
docker rm -f solonet
# Then retry
```

### 7. Block production takes long

**Symptom:** Waiting more than 3 minutes for first block.

**Context:** `wait-blockchain.sh` waits for 10 blocks to be produced. The script says
"Block production can take up to 3 minutes to start". This is normal for the first run.

**Diagnosis:**
```bash
docker exec solonet supervisorctl status
docker exec solonet tail -20 /var/log/monad-bft.log
docker exec solonet tail -20 /var/log/monad-execution.log
```

### 8. macOS: exec format error

**Symptom:** `exec format error` when running on Apple Silicon

**Root cause:** Image is amd64 only. Apple Silicon is ARM64.

**Solution:**
```bash
brew install lima colima lima-additional-guestagents
colima start --arch x86_64 --cpu-type max --cpu 8 --memory 16 --disk 300 --foreground
# Then in another terminal:
docker run --rm -it --privileged --network host \
  --ulimit nofile=16384:16384 \
  --name solonet monfops/monad-solonet
```

### 9. Privileged mode concerns

**Question:** "Is --privileged safe?"

**Answer:** --privileged is required for three reasons verified in the source code:
1. **Loop devices:** `prepare-disk.sh` uses `losetup` to create a loop device for TrieDB
2. **Hugepages:** `check-system.sh` writes to `/sys/.../hugepages/`
3. **Sysctl:** Tuning network buffers (`net.ipv4.tcp_rmem`, `net.core.rmem_max`, etc.)

This is safe for local development but should NEVER be used in production or on untrusted hosts.

### 10. Kernel too old: io_uring buffer ring crash (CONFIRMED)

**Symptom:** monad-bft enters a crash loop with:
```
thread 'monad-dataplane' panicked at 'failed to create buffer ring: Os { code: 22, kind: InvalidInput, message: "Invalid argument" }'
```

**Root cause:** Monad uses `IORING_REGISTER_PBUF_RING` (io_uring provided buffer rings),
which was added in Linux kernel 5.19. Monad v0.13.0 requires kernel >= 6.8.

**Verified on:** Hetzner dedicated server with Ubuntu 22.04, kernel 5.15.0. monad-bft
crashed every ~5 seconds and was restarted by supervisord in an infinite loop.

**Solution:** Use Ubuntu 24.04 with kernel 6.8+:
```bash
uname -r  # Must show >= 6.8
```

### 11. OOM kills on 4GB RAM (CONFIRMED)

**Symptom:** `monad-keystore` is killed during key generation. Container may exit or
get stuck with partially generated keys.

**Root cause:** `check-system.sh` sets `vm.nr_hugepages=2048`, which allocates 4GB
of 2MB hugepages. On a 4GB server, this leaves 0 bytes for processes.

**Verified on:** Hetzner CX23 (4GB RAM). After hugepages allocation, any process
requiring memory was killed by the OOM killer.

**Solution:** Use 8GB+ RAM. Tested successfully on Hetzner CX32 (8GB RAM).

### 12. Foundry (cast/forge) not found in PATH

**Symptom:**
```
OCI runtime exec failed: exec failed: unable to start container process: exec: "cast": executable file not found in $PATH
```

**Root cause:** Foundry is installed at `/root/.foundry/bin/` but this directory
is NOT in the default PATH for `docker exec`.

**Solution:**
```bash
docker exec solonet bash -lc 'PATH=/root/.foundry/bin:$PATH cast block-number'
# Or enter the container interactively:
docker exec -it solonet bash
export PATH=/root/.foundry/bin:$PATH
cast block-number
```

### 13. Multi-node: nodes stuck waiting

**Symptom:** `Waiting, have 1/3 ready` loop forever

**Root cause:** `build-config.sh` waits for `TOTAL_NODE_NUMBER` peer files in `/shared/peers/`.
If nodes don't share the same volume, they can't find each other.

**Solution:** All nodes must share the same `/shared` volume:
```bash
-v solonet-shared:/shared
```
And `TOTAL_NODE_NUMBER` must match across all nodes.

### 14. staking-cli: --private-key not accepted

**Symptom:**
```
error: unrecognized arguments: --private-key
```

**Root cause:** `staking-cli` does NOT accept `--private-key` on the command line.
It reads the private key from config file at `/solonet/config/staking-cli.toml`:
```toml
[staking]
funded_address_private_key = "0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d"
```

**Solution:** Use staking-cli without `--private-key`. To change the signing account, edit the config:
```bash
docker exec solonet sed -i 's/59c6995e.../ac0974bec.../' /solonet/config/staking-cli.toml
```

### 15. staking-cli: change-commission fails (Tx status: 0)

**Symptom:**
```
ERROR Transaction failed! Commission change was not successful for validator 1
```

**Root cause:** `change-commission` requires the validator's AuthAddress (Account #0: `0xf39Fd6...`),
but the default config uses Account #1's key. Only the validator owner can change commission.

**Solution:** Edit `/solonet/config/staking-cli.toml` to use Account #0's private key:
```toml
[staking]
funded_address_private_key = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
```

### 16. staking-cli: undelegate missing --withdrawal-id

**Symptom:**
```
error: the following arguments are required: --withdrawal-id
```

**Root cause:** `undelegate` requires `--withdrawal-id` which is not obvious. Each undelegation
gets a unique withdrawal ID used for later `withdraw`.

**Solution:**
```bash
staking-cli undelegate --validator-id 1 --amount 500 --withdrawal-id 1
```

### 17. staking-cli: withdraw — epoch delay

**Symptom:**
```
Cannot withdraw yet! Current epoch: 3, Need to wait until: 5
```

**Root cause:** This is correct behavior. Undelegated funds have an epoch delay (~2 epochs)
before they can be withdrawn. This is a security mechanism.

**Solution:** Wait for the required epoch. Check current epoch:
```bash
docker exec solonet staking-cli query epoch
```

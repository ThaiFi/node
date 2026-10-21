# ThaiFi Node

Guide to running a node on **ThaiFi Chain** (chain id **17**) — a blockchain forked from [Tempo](https://github.com/tempoxyz/tempo), maintained as **THBOS** ([ThaiFi/thbos](https://github.com/ThaiFi/thbos)), that uses **pathUSD** (6 decimals) as the fee token, with a 250 ms block time.

Node types covered by this guide: **full node / RPC follower** (not a validator — adding a validator requires approval from the ThaiFi team).

---

## 1. System Requirements

| Item | Minimum | Recommended |
|---|---|---|
| OS | Linux x86_64 (runs in Docker) | Ubuntu 22.04+ |
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 60 GB | 200 GB SSD/NVMe |
| Software | Docker + Docker Compose v2 | — |
| Ports | `30303` (TCP+UDP), `8545` (HTTP), `8546` (WS) | — |

✅ **Old CPUs**: the THBOS image is compiled with default x86-64 target flags — it runs on older CPUs without AVX2 (e.g. Intel Xeon X5650), tested. (This is why the compose file uses our own image instead of the upstream one, which is compiled with newer instruction sets and hits `SIGILL` on old CPUs.)

> ⚠️ **Disk type**: the node datadir is a heavy random-write workload (MDBX) — **SSD/NVMe is practically a requirement**, because the chain produces a block every 250 ms (4 blocks/s). Real-world testing on a 5400rpm HDD managed to execute only ~2 blocks/s, so it can never keep up with the tip (the gap keeps growing).
>
> ⚠️ **Avoid ZFS for the datadir**: reth detects and warns about this at startup — ZFS CoW interacts very poorly with MDBX; measured on the same workload, writes reach only ~4.5 MB/s on ZFS vs ~20 MB/s on ext4. A ZFS pool is fine for general storage, but keep the datadir on a separate ext4/xfs filesystem.

> ℹ️ **Image**: `ghcr.io/thaifi/thbos:v1.16.1-fee0.01` — ThaiFi's own build of the `v1.16.1` release plus the ThaiFi fee schedule ([source](https://github.com/ThaiFi/thbos/tree/thbos-v1.16.1)). If the bundled `tempo download` gives you trouble with the snapshot (see §7.1), report it to the ThaiFi team — we build and validate these images ourselves.

## 2. Quick Start (recommended — start from a snapshot)

Instead of syncing from genesis for hours, download the latest snapshot from ThaiFi (~3 GB, takes a few minutes):

```bash
# 1) Clone this repo (docker-compose.yml + genesis.json included)
git clone https://github.com/ThaiFi/node.git thaifi-node
cd thaifi-node

# 2) Create your own data dir + P2P key
mkdir -p data
printf '%s' "$(openssl rand -hex 32)" > data/discovery-secret

# 3) Download the latest snapshot (checksum is verified for you)
docker run --rm \
  -v $PWD/data:/data \
  -v $PWD/genesis.json:/config/genesis.json:ro \
  ghcr.io/thaifi/thbos:v1.16.1-fee0.01 \
  download \
  --manifest-url https://snapshots.thaifi.com/snapshots/current/manifest.json \
  --datadir /data \
  --chain /config/genesis.json \
  --force -y

# 4) Start the node
docker compose up -d
```

> Use `--full` instead of `-y` if you want complete data for every component (receipts, rocksdb indices — suited to explorer/archive nodes, ~2 GB extra download).

## 3. Verify the node is synced

```bash
# our node's block
curl -s -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

# chain head block (the two must match)
curl -s -X POST https://rpc.thaifi.com \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

The node follows the chain automatically via the public follow stream (`wss://ws.thaifi.com`) — check the logs with:

```bash
docker logs thaifi-node --tail 20
# You should normally see: Received new payload ... / Status connected_peers=N latest_block=...
```

**Check whether it keeps up with the tip:** run the two commands above and subtract the results — if the gap (distance from the public RPC) stays constant or shrinks to just a few blocks, you are synced. But if the gap **keeps growing**, the machine cannot execute blocks at the chain's 4 blocks/s rate — go back and review the disk requirements in §1.

**Transaction propagation (P2P):** the compose file already sets `--trusted-peers` to ThaiFi's 3 validators — your node receives txs/blocks directly from the validators.

## 4. Sync from genesis (alternative — no snapshot)

```bash
git clone https://github.com/ThaiFi/node.git thaifi-node && cd thaifi-node
mkdir -p data
printf '%s' "$(openssl rand -hex 32)" > data/discovery-secret
docker compose up -d
```

The node syncs the full pipeline from the trusted peers (takes several hours depending on the chain height).

## 5. Using the node

| Endpoint | URL |
|---|---|
| JSON-RPC (HTTP) | `http://localhost:8545` |
| JSON-RPC (WS) | `ws://localhost:8546` |

Add the network in MetaMask / any EIP-3085-capable wallet:

| Setting | Value |
|---|---|
| Chain ID | `17` (`0x11`) |
| RPC URL | `https://rpc.thaifi.com` |
| Currency | `pathUSD` (6 decimals) |
| Explorer | `https://exp.thaifi.com` |

**Tempo-based chain notes:**
- fees/gas are paid in **pathUSD** (`0x20c0000000000000000000000000000000000000`, 6 decimals) — not native ETH
- every TIP-20 token uses **6 decimals** (not 18 like typical ERC-20)
- **base fee ≈ 0.01 USD per standard transfer** (since 2026-10-21) — rises toward 0.20 USD automatically under congestion

## 6. Snapshots

| URL | Description |
|---|---|
| `https://snapshots.thaifi.com/snapshots/current/manifest.json` | latest snapshot (always use this one) |
| `https://snapshots.thaifi.com/snapshots/<YYYYMMDD-HHMM>/manifest.json` | individual rounds (kept as history) |
| `https://snapshots.thaifi.com/` | page listing all files |

Snapshots are generated from a quiesced (consistent) follower node and include blake3 checksums for every file in `manifest.json` — the `tempo download` command verifies them automatically.

### Creating/hosting your own snapshots

On an already-synced node:

```bash
# Always stop the node first (the database must be quiesced for a consistent snapshot)
docker compose stop

mkdir -p snapshot-output
docker run --rm \
  -v $PWD/data:/data \
  -v $PWD/genesis.json:/genesis.json:ro \
  -v $PWD/snapshot-output:/output \
  ghcr.io/thaifi/thbos:v1.16.1-fee0.01 \
  snapshot-manifest \
  --source-datadir /data \
  --consensus.datadir /data/consensus \
  --output-dir /output \
  --chain /genesis.json \
  --chain-id 17

docker compose up -d   # bring the node straight back up, then upload at your leisure
# Upload everything in snapshot-output/ to an S3/R2/HTTP server and point --manifest-url at manifest.json
```

## 7. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `connected_peers=0` for unusually long | firewall blocking `30303` TCP/UDP — open both outbound and inbound |
| `Illegal instruction` (SIGILL) when running the container | you are running an upstream `tempoxyz` image (compiled with newer instruction sets) — use the THBOS image `ghcr.io/thaifi/thbos:v1.16.1-fee0.01` instead, which runs on old CPUs |
| self-built binary reports `GLIBC_2.38 not found` / `CXXABI_1.3.15 not found` | wrong build base — you must use the `rust:1-bookworm` container (glibc matching the tempo image); if the same dir was previously built with a different distro, `rm -rf target` first, see **§7.2** |
| node stuck at the `Prune` stage, logging `pruned=0` thousands of times per second, `eth_blockNumber` frozen at the snapshot block | **prune loop** — seen with datadirs restored from a snapshot; fix in **§7.1** |
| node runs but the gap vs the public RPC keeps growing | the machine cannot execute 4 blocks/s — meet the §1 specs (SSD/NVMe) and make sure the datadir is on ext4/xfs, not ZFS |
| `download` reports `Server did not return file size` | the manifest/CDN endpoint doesn't send `Content-Length` — use the official URL `https://snapshots.thaifi.com/...` or check that your own reverse proxy sends the full headers |
| node synced but blocks not advancing | verify `--follow` points to `wss://ws.thaifi.com` and there are no consensus errors in the logs; try `docker compose up -d --force-recreate` |
| port conflicts with another service | change `8545/8546/30303/30306/8551` in `docker-compose.yml` |
| corrupted datadir (`database is corrupted`) | **never copy/move the datadir while the node is running** — always stop the node first; if corrupted, delete `data/` and restore from a fresh snapshot |

### 7.1 Node stuck at the `Prune` stage — endless `pruned=0` log loop (seen with snapshot-restored datadirs)

Symptoms: the node starts up normally, but `eth_blockNumber` stays frozen at the snapshot block forever, while the log floods with this message thousands of times per second along with nonstop disk writes:

```
sync::stages::prune::exec: Last segment has more data to prune last_segment=SenderRecovery progress=Finished pruned=0
reth_node_events::node: Committed stage progress pipeline_stages=13/14 stage=Prune checkpoint=4889950 target=5017965
```

Root cause: `reth.toml` in the datadir defines `[prune.segments]`, but the pruner can't find a prune target in the snapshot-restored data, so it loops deleting 0 rows and the `Prune` stage never finishes. You can confirm it with:

```bash
docker logs thaifi-node 2>&1 | grep -c "pruned=0"                        # keeps increasing = this is it
docker logs thaifi-node 2>&1 | grep -oE "checkpoint=[0-9]+" | tail -1    # frozen at the same value = looping
```

Fix — disable the prune segments (the node keeps full history and uses more disk, but for this chain it's still tiny):

```bash
docker compose stop
cd data                      # or your actual datadir path
cp reth.toml reth.toml.bak   # always back up first
# delete everything from [prune.segments] up to (but not including) [peers], then restore an empty section
sed -i '/^\[prune\.segments\]/,/^\[peers\]/{/^\[peers\]/!d}' reth.toml
sed -i '/^\[peers\]/i [prune.segments]\n' reth.toml
cd .. && docker compose up -d
```

After the fix, the pipeline resumes from the previous checkpoint immediately (no snapshot re-download needed), and the node starts following the tip as described in §3.

> Note: this issue affects datadirs restored from a snapshot — nodes synced from genesis from the start are not affected.

### 7.2 Building images for old CPUs (historical)

The THBOS images (`ghcr.io/thaifi/thbos:*`) are compiled with default x86-64 target flags and run on old CPUs — no special build needed. This section applied to the upstream `tempoxyz` images, which are compiled with newer instruction sets and hit `SIGILL` on pre-AVX2 CPUs even though `--version` works.

If you ever need to build the upstream image yourself for an old CPU: compile inside a `rust:1-bookworm` container (glibc must match the Debian 12 base image — `rust:latest` produces binaries that fail with `GLIBC_2.38 not found`), use `RUSTFLAGS='-C target-cpu=x86-64'`, `rm -rf target` if the dir was previously built with a different distro, and always test a real workload — `--version` alone is not enough because unsupported instructions only crash in the heavy runtime paths.

## 8. Chain Information

| Setting | Value |
|---|---|
| Chain ID | 17 |
| Type | Tempo fork — THBOS ([ThaiFi/thbos](https://github.com/ThaiFi/thbos)) · T12 active · **THBOS fee schedule active since 2026-10-21 19:40 ICT** (standard TIP-20 transfer = 0.01 USD) |
| Node image | `ghcr.io/thaifi/thbos:v1.16.1-fee0.01` ([branch `thbos-v1.16.1`](https://github.com/ThaiFi/thbos/tree/thbos-v1.16.1)) |
| Block time | 250 ms |
| Fee token | pathUSD `0x20c0000000000000000000000000000000000000` (6 dec) |
| Public RPC | `https://rpc.thaifi.com` |
| Follow WS | `wss://ws.thaifi.com` |
| Explorer | `https://exp.thaifi.com` |
| Snapshot | `https://snapshots.thaifi.com` |
| Contract verification | `https://contracts.thaifi.com` |
| Tokenlist | `https://tokenlist.thaifi.com/list/17` |

---

To become a **validator**, or if you hit connectivity issues, contact the ThaiFi team · [thaifi.com](https://thaifi.com)

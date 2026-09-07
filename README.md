# ThaiFi Node

Private Tempo chain + Zone deployment, based on [tempoxyz/tempo](https://github.com/tempoxyz/tempo) and [tempoxyz/zones](https://github.com/tempoxyz/zones).

## Structure (git submodules)

| Path | Repo | Ref |
|------|------|-----|
| `tempo/` | [ThaiFi/tempo](https://github.com/ThaiFi/tempo) (fork) | branch `feat/private-chain-genesis` |
| `zones/` | [tempoxyz/zones](https://github.com/tempoxyz/tempo) (upstream) | pinned to tested commit `48e63e37` |

The tempo fork carries private-chain genesis flags not (yet) upstream:

- `--zone-factory-owner <ADDR>` — custom ZoneFactory owner (upstream hardcodes `INITIAL_FACTORY_OWNER`)
- `--genesis-timestamp <UNIX>` — chain start time
- `--gas-token-name/symbol/currency` — customize the primary gas token

> **Note:** zones is pinned because the zone sequencer speaks the **T10** portal ABI.
> Upstream zones may move to T12; keep both sides consistent.

## Clone

```bash
git clone --recurse-submodules https://github.com/ThaiFi/node.git
cd node
```

## Build

```bash
# L1 node + genesis tooling
cd tempo && cargo build --release --bin tempo --bin tempo-xtask && cd ..

# Zone node + tooling
cd zones && cargo build --release --bin tempo-zone --bin tempo-xtask && cd ..
```

Prereqs: Rust (rustup), Foundry 1.8+ (`cast`, `forge`), `just`, `jq`.

## Genesis (3-validator private chain, chain ID 17)

```bash
cd tempo
./target/release/tempo-xtask generate-genesis \
  --chain-id 17 \
  --accounts 0 \
  --pathusd-amount 1000000 \
  --pathusd-admin <ADMIN_ADDR> \
  --no-extra-tokens \
  --no-pairwise-liquidity \
  --validators "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002" \
  --validator-addresses <V1_ADDR>,<V2_ADDR>,<V3_ADDR> \
  --zone-factory-owner <FACTORY_OWNER_ADDR> \
  --t11-time 9999999998 \
  --t12-time 9999999999 \
  --output <OUT_DIR>
```

Key points:

- `--t11-time/--t12-time` in the future keeps the **T10** ZonePortal runtime
  (the zone sequencer's `submitBatch` ABI matches T10, not T12).
- `--no-extra-tokens --no-pairwise-liquidity` = pathUSD-only minimal genesis.
  FeeAMM liquidity seeding needs ≥ 30,000 pathUSD held by `--pathusd-admin`;
  skip it (or raise the amount) for a minimal supply.
- Genesis generates `signing.key` + `signing.share` per validator under `<OUT_DIR>/<ip:port>/`.

## Run L1 validators (production mode, 3 nodes)

```bash
tempo node --chain <OUT_DIR>/genesis.json \
  --datadir <NODE_DATA_DIR> \
  --consensus.signing-key <OUT_DIR>/127.0.0.1:3000/signing.key \
  --consensus.signing-share <OUT_DIR>/127.0.0.1:3000/signing.share \
  --consensus.listen-address 127.0.0.1:3000 \
  --consensus.use-local-defaults \
  --http --http.port 8545 --ws --ws.port 8546 \
  --authrpc.port 8551 --port 30303 --disable-discovery
# nodes 2/3: unique HTTP/WS/auth/P2P ports + their own key/share/listen-address
```

`--consensus.use-local-defaults` is required on a local/private network:
Commonware P2P refuses to dial private IPs (127.0.0.1, RFC1918) otherwise.

## Deploy a zone

```bash
# on L1: create the zone (requires the ZoneFactory owner key)
cast send 0x5af2000000000000000000000000000000000000 \
  "createZone((address,bool,bool,address[],address[],address,address[],uint8,string))" \
  "(<TOKEN>,false,false,[],[],<ZONE_ADMIN>,[<SEQUENCER>],1,<ZONE_RPC_URL>)" \
  --private-key <FACTORY_OWNER_KEY> --gas-limit 30000000 --legacy

# register the sequencer ECIES encryption key (enables deposits)
zones/target/release/tempo-xtask set-encryption-key \
  --l1-rpc-url <L1_HTTP> --portal <PORTAL_ADDR> --private-key <SEQUENCER_KEY>

# generate zone genesis (zone chain ID = (l1_chain_id << 32) | zone_id)
zones/target/release/tempo-xtask generate-zone-genesis \
  --output <ZONE_DIR> --chain-id <ZONE_CHAIN_ID> \
  --admin <ZONE_ADMIN> --sequencer <SEQUENCER> \
  --l1-rpc-url <L1_HTTP> --tempo-portal <PORTAL_ADDR>

# run the zone sequencer
zones/target/release/tempo-zone node \
  --chain <ZONE_DIR>/genesis.json --datadir <ZONE_DATA> \
  --http --http.port 9545 --ws --ws.port 9546 \
  --l1.rpc-url ws://<L1_WS> --l1.portal-address <PORTAL_ADDR> \
  --sequencer-key-file <SEQUENCER_KEY_FILE> --sequencer
```

Funding notes: accounts need **pathUSD** (fee token), not native ETH.
Deposits require `approve(portal)` on L1 + `set-encryption-key`;
withdrawals require `approve(0x1c00...0002 outbox)` on the zone.
Zone TIP-20 `transfer()` is intentionally disabled (permissioned phase).

## Update submodules

```bash
git submodule update --remote tempo   # follow feat/private-chain-genesis
cd zones && git fetch && git checkout <new-sha> && cd ..
git commit -am "chore: bump submodules"
```

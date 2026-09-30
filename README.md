# ThaiFi Node

คู่มือเปิด node ของ **ThaiFi Chain** (chain id **17**) — blockchain ที่ fork จาก [Tempo](https://github.com/tempoxyz/tempo) ใช้ fee token เป็น **pathUSD** (6 decimals) และ block time 250ms

Node ประเภทที่คู่มือนี้รองรับ: **full node / RPC follower** (ไม่ใช่ validator — การเพิ่ม validator ต้องได้รับอนุมัติจากทาง ThaiFi)

---

## 1. ความต้องการของระบบ

| รายการ | ขั้นต่ำ | แนะนำ |
|---|---|---|
| OS | Linux x86_64 (ทำงานบน Docker) | Ubuntu 22.04+ |
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 60 GB | 200 GB SSD/NVMe |
| ซอฟต์แวร์ | Docker + Docker Compose v2 | — |
| Ports | `30303` (TCP+UDP), `8545` (HTTP), `8546` (WS) | — |

> ⚠️ **CPU เก่า**: image ทางการ compile ด้วย instruction set ใหม่ — เครื่องที่ CPU ไม่มี AVX2 (เช่น Intel รุ่นก่อน Sandy Bridge) จะเจอ `SIGILL` ตอนรันจริง (ทั้งที่ `--version` ผ่าน) ต้อง build จาก source เองด้วย `RUSTFLAGS="-C target-cpu=x86-64"` ดู §7

## 2. Quick Start (แนะนำ — เริ่มจาก snapshot)

แทนการ sync จาก genesis หลายชั่วโมง ให้ดาวน์โหลด snapshot ล่าสุดจาก ThaiFi (~3 GB, ใช้เวลาไม่กี่นาที):

```bash
# 1) โคลน repo นี้ (มี docker-compose.yml + genesis.json พร้อม)
git clone https://github.com/ThaiFi/node.git thaifi-node
cd thaifi-node

# 2) สร้าง data dir + P2P key ของตัวเอง
mkdir -p data
printf '%s' "$(openssl rand -hex 32)" > data/discovery-secret

# 3) ดาวน์โหลด snapshot ล่าสุด (verify checksum ให้เอง)
docker run --rm \
  -v $PWD/data:/data \
  -v $PWD/genesis.json:/config/genesis.json:ro \
  ghcr.io/tempoxyz/tempo:latest \
  download \
  --manifest-url https://snapshots.thaifi.com/snapshots/current/manifest.json \
  --datadir /data \
  --chain /config/genesis.json \
  --force -y

# 4) เปิด node
docker compose up -d
```

> ใช้ `--full` แทน `-y` ถ้าต้องการข้อมูลครบทุก component (receipts, rocksdb indices — เหมาะกับ node ที่ทำ explorer/archive, โหลดเพิ่ม ~2 GB)

## 3. ตรวจสอบว่า node sync แล้ว

```bash
# block ของ node เรา
curl -s -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

# block head ของ chain (ต้องตรงกัน)
curl -s -X POST https://rpc.thaifi.com \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

node จะ follow chain ผ่าน public follow stream (`wss://rpc.thaifi.com/ws`) อัตโนมัติ — ตรวจ log ได้ด้วย:

```bash
docker logs thaifi-node --tail 20
# ปกติจะเห็น: Received new payload ... / Status connected_peers=N latest_block=...
```

**การแพร่กระจายธุรกรรม (P2P):** compose ตั้ง `--trusted-peers` ไว้ที่ validator 3 ตัวของ ThaiFi ให้แล้ว — node เราจะรับ tx/block จาก validator ตรง

## 4. Sync จาก genesis (ทางเลือก — ไม่โหลด snapshot)

```bash
git clone https://github.com/ThaiFi/node.git thaifi-node && cd thaifi-node
mkdir -p data
printf '%s' "$(openssl rand -hex 32)" > data/discovery-secret
docker compose up -d
```

node จะ sync pipeline ทั้งหมดจาก trusted-peers (ใช้เวลาหลายชั่วโมงขึ้นกับความสูงของ chain)

## 5. ใช้งาน node

| Endpoint | URL |
|---|---|
| JSON-RPC (HTTP) | `http://localhost:8545` |
| JSON-RPC (WS) | `ws://localhost:8546` |

เพิ่มเครือข่ายใน MetaMask / wallet ที่รองรับ EIP-3085:

| ค่า | ค่า |
|---|---|
| Chain ID | `17` (`0x11`) |
| RPC URL | `https://rpc.thaifi.com` |
| Currency | `pathUSD` (6 decimals) |
| Explorer | `https://exp.thaifi.com` |

**ข้อควรรู้ของ Tempo-based chain:**
- fee/gas จ่ายด้วย **pathUSD** (`0x20c0000000000000000000000000000000000000`, 6 decimals) — ไม่ใช่ native ETH
- TIP-20 token ทุกตัวใช้ **6 decimals** (ไม่ใช่ 18 แบบ ERC-20 ทั่วไป)

## 6. Snapshots

| URL | ความหมาย |
|---|---|
| `https://snapshots.thaifi.com/snapshots/current/manifest.json` | snapshot ล่าสุด (ใช้ตัวนี้เสมอ) |
| `https://snapshots.thaifi.com/snapshots/<YYYYMMDD-HHMM>/manifest.json` | snapshot แต่ละรอบ (เก็บเป็นประวัติ) |
| `https://snapshots.thaifi.com/` | หน้ารวมไฟล์ทั้งหมด |

snapshot ถูกสร้างจาก follower node ที่หยุดนิ่ง (consistent) และมี blake3 checksum ของทุกไฟล์ใน `manifest.json` — คำสั่ง `tempo download` ตรวจให้เอง

### อยากสร้าง/โฮสต์ snapshot เอง

บน node ที่ sync แล้ว:

```bash
# หยุด node ก่อนเสมอ (database ต้องนิ่งจึงได้ snapshot ที่ consistent)
docker compose stop

mkdir -p snapshot-output
docker run --rm \
  -v $PWD/data:/data \
  -v $PWD/genesis.json:/genesis.json:ro \
  -v $PWD/snapshot-output:/output \
  ghcr.io/tempoxyz/tempo:latest \
  snapshot-manifest \
  --source-datadir /data \
  --consensus.datadir /data/consensus \
  --output-dir /output \
  --chain /genesis.json \
  --chain-id 17

docker compose up -d   # เปิด node คืนทันที แล้วค่อยอัปโหลด
# อัปโหลดไฟล์ทั้งหมดใน snapshot-output/ ไป S3/R2/HTTP server แล้วชี้ --manifest-url ที่ manifest.json
```

## 7. Troubleshooting

| อาการ | สาเหตุ/วิธีแก้ |
|---|---|
| `connected_peers=0` นานผิดปกติ | firewall บล็อก `30303` TCP/UDP — เปิด outbound + inbound ให้ครบ |
| `Illegal instruction` (SIGILL) เมื่อรัน container | CPU เก่าไม่มี AVX2 — build image จาก source: `git clone https://github.com/tempoxyz/tempo` ที่ commit เดียวกับ image ที่ chain ใช้ แล้ว `RUSTFLAGS="-C target-cpu=x86-64" cargo build --release --bin tempo` แล้วแทนที่ `/usr/local/bin/tempo` ใน image |
| `download` ฟ้อง `Server did not return file size` | ปลายทาง manifest/CDN ไม่ส่ง `Content-Length` — ใช้ URL ทางการ `https://snapshots.thaifi.com/...` หรือตรวจ reverse proxy ของตัวเองว่าส่ง header ครบ |
| node sync แล้วแต่ block ไม่เดิน | ตรวจว่า `--follow` ชี้ `wss://rpc.thaifi.com/ws` และ log ไม่มี error ฝั่ง consensus; ลอง `docker compose up -d --force-recreate` |
| port ชนกับ service อื่น | เปลี่ยน `8545/8546/30303/30306/8551` ใน `docker-compose.yml` |
| datadir เสียหาย (`database is corrupted`) | **ห้าม copy/move datadir ขณะ node รันอยู่** — หยุด node ก่อนเสมอ; กรณีเสียหายให้ลบ `data/` แล้ว restore จาก snapshot ใหม่ |

## 8. ข้อมูล Chain

| ค่า | ค่า |
|---|---|
| Chain ID | 17 |
| ประเภท | Tempo fork (T11 active) |
| Block time | 250 ms |
| Fee token | pathUSD `0x20c0000000000000000000000000000000000000` (6 dec) |
| Public RPC | `https://rpc.thaifi.com` |
| Follow WS | `wss://rpc.thaifi.com/ws` |
| Explorer | `https://exp.thaifi.com` |
| Snapshot | `https://snapshots.thaifi.com` |
| Contract verification | `https://contracts.thaifi.com` |
| Tokenlist | `https://tokenlist.thaifi.com/list/17` |

---

หากต้องการเป็น **validator** หรือพบปัญหาการเชื่อมต่อ ติดต่อทีม ThaiFi · [thaifi.com](https://thaifi.com)

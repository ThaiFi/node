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

> ⚠️ **CPU เก่า**: image ทางการ compile ด้วย instruction set ใหม่ — เครื่องที่ CPU ไม่มี AVX2 (เช่น Intel รุ่นก่อน Sandy Bridge) จะเจอ `SIGILL` ตอนรันจริง (ทั้งที่ `--version` ผ่าน) ต้อง build จาก source เองด้วย `RUSTFLAGS="-C target-cpu=x86-64"` ดู §7.2

> ⚠️ **ประเภทดิสก์**: datadir ของ node คือ workload random-write หนัก (MDBX) — **SSD/NVMe เกือบจำเป็น** เพราะ chain ผลิต block ทุก 250 ms (4 blocks/s) การทดสอบจริงบน HDD 5400rpm execute ได้เพียง ~2 blocks/s จึงไม่มีวันตาม tip ทัน (gap ขยายตัวเรื่อย ๆ)
>
> ⚠️ **หลีกเลี่ยง ZFS สำหรับ datadir**: reth จะตรวจจับและเตือนเองตอน startup — CoW ของ ZFS เข้ากับ MDBX ได้แย่มาก วัดจริงงานเดียวกันเขียนได้ ~4.5 MB/s บน ZFS เทียบกับ ~20 MB/s บน ext4 ถ้ามี ZFS pool ไว้เก็บของทั่วไปก็ดี แต่ให้วาง datadir ไว้บน ext4/xfs แยกต่างหาก

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

node จะ follow chain ผ่าน public follow stream (`wss://ws.thaifi.com`) อัตโนมัติ — ตรวจ log ได้ด้วย:

```bash
docker logs thaifi-node --tail 20
# ปกติจะเห็น: Received new payload ... / Status connected_peers=N latest_block=...
```

**เช็คว่าตาม tip ทันหรือไม่:** รันสองคำสั่งข้างบนแล้วแปลงผลมาลบกัน — ถ้า gap (ห่างจาก public RPC) คงที่หรือหดจนเหลือไม่กี่ block แปลว่า sync ทัน แต่ถ้า gap **ขยายตัวต่อเนื่อง** แปลว่าเครื่อง execute block ไม่ทันอัตรา 4 blocks/s ของ chain ให้กลับไปดูข้อกำหนดดิสก์ใน §1

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
| `Illegal instruction` (SIGILL) เมื่อรัน container | CPU เก่าไม่มี AVX2 — build image จาก source เอง ดูสเต็ปเต็มและกับดักที่พบจริงใน **§7.2** |
| build เองแล้ว binary ฟ้อง `GLIBC_2.38 not found` / `CXXABI_1.3.15 not found` | build ผิด base — ต้องใช้ container `rust:1-bookworm` (glibc ตรงกับ image ของ tempo) และถ้าเคย build ด้วย distro อื่นใน dir เดิม ต้อง `rm -rf target` ก่อน ดู **§7.2** |
| node ค้างที่ stage `Prune`, log `pruned=0` เป็นพันรอบ/วินาที, `eth_blockNumber` นิ่งที่ block ของ snapshot | **prune loop** — พบกับ datadir ที่ restore จาก snapshot วิธีแก้ดู **§7.1** |
| node เดินแต่ gap กับ public RPC ขยายตัวต่อเนื่อง | เครื่อง execute ไม่ทัน 4 blocks/s — ตามสเปกใน §1 (SSD/NVMe) และย้ำว่า datadir ต้องอยู่บน ext4/xfs ไม่ใช่ ZFS |
| `download` ฟ้อง `Server did not return file size` | ปลายทาง manifest/CDN ไม่ส่ง `Content-Length` — ใช้ URL ทางการ `https://snapshots.thaifi.com/...` หรือตรวจ reverse proxy ของตัวเองว่าส่ง header ครบ |
| node sync แล้วแต่ block ไม่เดิน | ตรวจว่า `--follow` ชี้ `wss://ws.thaifi.com` และ log ไม่มี error ฝั่ง consensus; ลอง `docker compose up -d --force-recreate` |
| port ชนกับ service อื่น | เปลี่ยน `8545/8546/30303/30306/8551` ใน `docker-compose.yml` |
| datadir เสียหาย (`database is corrupted`) | **ห้าม copy/move datadir ขณะ node รันอยู่** — หยุด node ก่อนเสมอ; กรณีเสียหายให้ลบ `data/` แล้ว restore จาก snapshot ใหม่ |

### 7.1 Node ค้างที่ stage `Prune` — log `pruned=0` วนไม่จบ (พบกับ datadir จาก snapshot)

อาการ: node รันขึ้นได้ตามปกติ แต่ `eth_blockNumber` ค้างที่ block ของ snapshot ตลอดไป ขณะที่ log เอ่อล้นด้วยข้อความนี้หลายพันรอบต่อวินาที พร้อมเขียนดิสก์ไม่หยุด:

```
sync::stages::prune::exec: Last segment has more data to prune last_segment=SenderRecovery progress=Finished pruned=0
reth_node_events::node: Committed stage progress pipeline_stages=13/14 stage=Prune checkpoint=4889950 target=5017965
```

ต้นตอ: `reth.toml` ใน datadir กำหนด `[prune.segments]` ไว้ แต่ pruner หา prune target ในข้อมูลที่ restore จาก snapshot ไม่เจอ จึงวนลูปลบ 0 แถว และ stage `Prune` ไม่มีวันเสร็จ — เช็คยืนยันได้ว่า:

```bash
docker logs thaifi-node 2>&1 | grep -c "pruned=0"                        # เพิ่มขึ้นเรื่อย ๆ = ใช่
docker logs thaifi-node 2>&1 | grep -oE "checkpoint=[0-9]+" | tail -1    # นิ่งตัวเดิมตลอด = วนลูป
```

วิธีแก้ — ปิด prune segments (node จะเก็บ history เต็ม ใช้ดิสก์มากขึ้น แต่สำหรับ chain นี้ยังเล็กมาก):

```bash
docker compose stop
cd data                      # หรือ path ของ datadir จริง
cp reth.toml reth.toml.bak   # backup เสมอ
# ลบทิ้งตั้งแต่ [prune.segments] จนถึงก่อน [peers] แล้วคืน section เปล่ากลับมา
sed -i '/^\[prune\.segments\]/,/^\[peers\]/{/^\[peers\]/!d}' reth.toml
sed -i '/^\[peers\]/i [prune.segments]\n' reth.toml
cd .. && docker compose up -d
```

หลังแก้ pipeline จะไล่ stage ต่อจาก checkpoint เดิมทันที (ไม่ต้องโหลด snapshot ใหม่) แล้ว node จะเริ่ม follow tip ได้ตาม §3

> หมายเหตุ: อาการนี้พบกับ datadir ที่ restore จาก snapshot — node ที่ sync จาก genesis ตั้งแต่ต้นไม่พบ

### 7.2 Build image เองสำหรับ CPU ไม่มี AVX2

ตรวจก่อน: `grep -q avx2 /proc/cpuinfo && echo HAS_AVX2 || echo NO_AVX2`

ถ้า `NO_AVX2` — official image จะ `--version` ผ่าน แต่ crash (`exit code 132` / SIGILL) ตอนทำงานจริง เช่น ตอนรัน `tempo download` หรือ node ต้อง compile เองดังนี้:

```bash
# 1) โคลน tempo ที่ commit เดียวกับ image (ดู Commit SHA จาก:
#    docker run --rm --entrypoint tempo ghcr.io/tempoxyz/tempo:latest --version)
git clone https://github.com/tempoxyz/tempo && cd tempo
git fetch --depth 1 origin <COMMIT_SHA> && git checkout FETCH_HEAD

# 2) build ใน container rust:1-bookworm ให้ glibc ตรงกับ base image ของ tempo (Debian 12 / glibc 2.36)
#    ใช้เวลา ~30-60 นาที
docker run -d --name tempo-build \
  -v $PWD:/repo -w /repo rust:1-bookworm \
  bash -c "apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y clang libclang-dev cmake pkg-config && \
           RUSTFLAGS='-C target-cpu=x86-64' cargo build --release --bin tempo && \
           cp target/release/tempo /repo/tempo-x86-64 && echo BUILD_DONE"
docker logs -f tempo-build

# 3) patch binary ลง image แล้วเปลี่ยน image ใน docker-compose.yml เป็น tempo:x86-64
printf 'FROM ghcr.io/tempoxyz/tempo:latest\nCOPY tempo-x86-64 /usr/local/bin/tempo\n' > Dockerfile.patch
docker build -f Dockerfile.patch -t tempo:x86-64 .
docker run --rm tempo:x86-64 --version   # smoke test
```

กับดักที่เจอจริง (รวมเป็น 3 รอบ build กว่าจะผ่าน):

- **ห้ามใช้ `rust:latest`** (ปัจจุบันเป็น trixie / glibc 2.41) — binary ที่ได้จะฟ้อง `GLIBC_2.38 not found` บน base image ของ tempo ให้ใช้ `rust:1-bookworm` เสมอ
- **ถ้า dir เดิมเคย build ด้วย distro อื่น ต้อง `rm -rf target` ก่อน build ใหม่** — cache ของ native artifacts (เช่น librocksdb.a) จำ fingerprint ของ compiler แล้วปล่อย object ที่ผูกกับ glibc ใหม่ไป link ซ้ำ ทำให้ยังเจอ error เดิมแม้เปลี่ยนมา build บน bookworm แล้ว
- `--version` ผ่าน**ยังไม่พอ** — ต้องลองงานจริง (เช่น `tempo download ... --manifest-url ...`) เพราะ instruction ที่ CPU ไม่รองรับจะไป crash ตอน runtime path ที่รันหนัก ๆ เท่านั้น

## 8. ข้อมูล Chain

| ค่า | ค่า |
|---|---|
| Chain ID | 17 |
| ประเภท | Tempo fork (T11 active) |
| Block time | 250 ms |
| Fee token | pathUSD `0x20c0000000000000000000000000000000000000` (6 dec) |
| Public RPC | `https://rpc.thaifi.com` |
| Follow WS | `wss://ws.thaifi.com` |
| Explorer | `https://exp.thaifi.com` |
| Snapshot | `https://snapshots.thaifi.com` |
| Contract verification | `https://contracts.thaifi.com` |
| Tokenlist | `https://tokenlist.thaifi.com/list/17` |

---

หากต้องการเป็น **validator** หรือพบปัญหาการเชื่อมต่อ ติดต่อทีม ThaiFi · [thaifi.com](https://thaifi.com)

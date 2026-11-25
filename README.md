# FLOP Labs Testnet: A Beginner's Guide for Validators

**The FLOP testnet is live** (opened 24 November 2026) and registration is open for the **first 50
validator seats**. This guide walks a complete beginner through FLOP's official preparation steps,
with every command to copy and paste.

Last checked against FLOP's official pages: **25 Nov 2026**.

> **Unofficial community guide.** It follows FLOP's official
> [validator prep kit](https://docs.flop.network/testnet-validator-prep/),
> [Testnet overview](https://docs.flop.network/networks/testnet) and
> [testnet page](https://flop.finance/testnet/). Things change daily during the opening: always check
> those pages and the [Testnet Operator Hub](https://docs.flop.network/operators?network=testnet) first.

---

## Part 0. What's happening (read in 3 minutes)

| | |
|---|---|
| **Status** | Live, **unscored opening** since 24 Nov 2026. Six founding validators produce and finalize blocks |
| **Seats** | Registration open for the **first 50 validator seats**. FLOP's target: 50 active validators, then about 25% more per day as applicants qualify |
| **Test FLOP** | Has **no monetary value**. Expect outages, upgrades and possible resets |
| **What counts** | A later **scored phase** (up to ~90 days) is the only route to the genesis airdrop. Validators are **ranked on-chain** on performance, and mainnet validator admission is by testnet ranking |
| **Cost** | The stake for testnet validators can be **sponsored by the network**, so your cost is the server and your time |
| **Mainnet** | Planned Q1 2027 |

**What you can do today, in order:**

1. Check the live testnet yourself
2. Register your team (takes 5 minutes, no server needed)
3. Get the right server and fill in FLOP's host worksheet
4. Install the tools
5. Rehearse on devnet: run a keyless node and record proof
6. Update your registration
7. When FLOP publishes the signed testnet kit: verify it and join

**What you can't do yet:** run a testnet validator. The **signed raw chain spec and Linux node kit
are pending publication**. Don't generate keys or sign anything for testnet until they're out.

> ⚠️ **Scam warning.** There is no paid seat, no token sale and no "guaranteed spot". FLOP never asks
> for your seed phrase, private keys or access to your server, and no payment or signature is due to register.

---

## Part 1. See the live testnet yourself

From any computer with a terminal:

```bash
curl -fsS https://testnet.flop.network/ -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"chain_getBlockHash","params":[0]}'
```

The `result` must be the official testnet genesis:

```text
0x918abd58fb16306b30ef3886514cfaebffecb4de43e82b6f8ec4795c43cfd800
```

You can also watch blocks on the explorer: **[testnet.flopscan.io](https://testnet.flopscan.io/)**.

---

## Part 2. Register your team (do this now)

You can register **before** your server is ready.

1. Create a free **GitHub** account if you don't have one.
2. Open the official
   **[operator registration form](https://github.com/flop-labs/network/issues/new?template=operator-application.yml)**.
3. Fill it in:

| Field | What to write |
|---|---|
| Team ID | A public team name you'll keep using |
| Network | **testnet** |
| Requested role | **validator candidate** |
| Existing application | Link your devnet application if you have one, else leave empty |
| Planned capacity and host | Planned number of hosts, provider and region, CPU / RAM / disk / OS |
| Preparation stage | Be honest, e.g. **planning a host** (you'll update it later) |
| Sync and restart evidence | Leave empty for now (Part 6) |
| Notes | Any blockers, and ask for a **private coordination channel** for contacts and detailed evidence |

4. Submit. Your **issue number is your application ID**.

Rules:

- **One issue per team per network.** Already registered? Update that issue instead of opening a new one.
- The issue is **public**: never paste seeds, private keys, `account.json`, `.env`, keystores or tokens.
- Registering records interest. Seat assignment, sponsorship and activation are separate later steps.

---

## Part 3. Get the right server

### 3.1 Server size

FLOP's official sizing:

| | Production sizing reference (prep kit) | Recommended (testnet page, provisional) |
|---|---|---|
| CPU | **8 physical cores, 3.4 GHz or faster** | 8+ cores |
| RAM | **32 GB ECC** (64 GB recommended) | 64 GB |
| Disk | **4 TB enterprise NVMe** | 4 TB NVMe (2 TB minimum) |
| Network | **1 Gbps symmetric** | 1 Gbps redundant |

Choose **Ubuntu 24.04 LTS** on **x86-64 (amd64)**, with a stable **public IPv4**. Use a
**dedicated** server that runs nothing else. Record the capacity you *actually* have; that's what
FLOP asks for.

> Just rehearsing first (Part 5)? A smaller server is fine for the devnet rehearsal alone:
> FLOP's pilot reference is 4 vCPU, 16 GB RAM, 300 GB SSD.

### 3.2 Fill in FLOP's host worksheet (keep it private)

Download FLOP's blank worksheet:

```bash
curl -fLO https://docs.flop.network/testnet-validator-prep/host-worksheet.csv
```

Open it in any spreadsheet app. Use **one row per proposed testnet node**: operator and controlling
entity, shared infrastructure, on-call contacts, time zone, provider, ASN, region, CPU, RAM, free
NVMe, OS, bandwidth, planned public P2P and DA addresses, accounts, and whether you'll self-fund or
request sponsorship. Mark unknowns as `pending`.

**Keep your filled copy private.** Share it only through the private channel FLOP sets up with you.

---

## Part 4. Connect to your server and install the tools

### 4.1 Log in

**Windows:** open PowerShell. **Mac / Linux:** open Terminal.

```bash
ssh root@YOUR_SERVER_IP
```

Everything below runs **on the server**.

### 4.2 Tools FLOP requires: Linux amd64, Node.js 24+, Docker Engine 28+, Compose, flock

```bash
apt update && apt upgrade -y
apt install -y curl jq ufw zstd util-linux
curl -fsSL https://get.docker.com | sh
curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
apt install -y nodejs
docker --version && docker compose version && node --version && flock --version
```

Check: Docker **28 or newer**, Node **v24**, and all four print a version.

### 4.3 Firewall: the ports FLOP plans

```bash
ufw allow OpenSSH
ufw allow 30333/tcp   # P2P: other nodes connect to yours
ufw allow 7001/tcp    # filtered data availability (DA)
ufw --force enable
ufw status
```

Open the same two ports in your **hosting provider's** firewall too. **Never** expose node RPC
**9944**, metrics **9615** or raw DA **7000** to the internet.

---

## Part 5. Rehearse on devnet: run a keyless node and record proof

This is FLOP's official step 2 ("prepare one host and rehearse on devnet"). A **keyless node**
follows the chain but holds no keys and signs nothing. It proves your server can sync, agree with
FLOP's chain and survive a restart. It runs on **devnet**, because the testnet kit isn't public yet.
Keep it completely separate from any future testnet data.

All commands come from FLOP's official
[Devnet RPC guide](https://gist.github.com/sv/7c8b93f3868bc15e3b6eabe0578057e3#run-a-flop-devnet-rpc-replica).

### 5.1 Download FLOP's files and check they're genuine

Paste the whole block. **If any line doesn't end in `OK`, stop.**

```bash
set -euo pipefail
mkdir -p "$HOME/flop-devnet-rpc"
cd "$HOME/flop-devnet-rpc"
files=https://gist.githubusercontent.com/sv/7c8b93f3868bc15e3b6eabe0578057e3/raw/21adaa99c601003278d1682bf937696f21d31432
for file in docker-compose.rpc.yml node-entrypoint.sh rpc_ready.py rpc-pool.env.example rpc-origin.Caddyfile; do
  curl --fail --location --output "$file" "$files/$file"
done
sha256sum --check <<'SUMS'
6487cd84d5a257659297ad7b8fdd55978e6d8327bfe029d9460645da2591345d  docker-compose.rpc.yml
7279e81c0d6de4b6c59733b64132f929b8b7dd5825b8f37433d09940a93f983d  node-entrypoint.sh
6a066b7540628c7db4f455d7bfd2540db54355a4c0a2959d97032b76bcf641cd  rpc_ready.py
4f7295735f75b5a7b4c0babe3be4b0eef687decab99ba1612f91ee6f3b04d1bb  rpc-pool.env.example
e184d4f38f054dbe7947efde4d85e6a8964f02e594666ad42b9216dd98a4b2d5  rpc-origin.Caddyfile
SUMS
curl --fail --location --output chain.raw.json \
  https://explorer.devnet.flop.network/validator-downloads/flop_devnet_ts-99c172c095340ace.raw.json
printf '%s  %s\n' \
  3b213c38e7678285bc38fbdfded5dbe3d6acbd16e836b14aaaeef2f6ae1cd16e \
  chain.raw.json | sha256sum --check
cp -n rpc-pool.env.example rpc-pool.env
```

### 5.2 Set your IP addresses

```bash
hostname -I | awk '{print $1}'           # LOCAL IP  -> HOST_IP
curl -4 -s https://api.ipify.org; echo   # PUBLIC IP -> PUBLIC_ADDR
nano rpc-pool.env
```

Fill in these two lines (leave everything else as it is):

```text
HOST_IP=YOUR_LOCAL_IP
PUBLIC_ADDR=/ip4/YOUR_PUBLIC_IP/tcp/30333
```

Make sure these stay on localhost:

```text
RPC_BIND_IP=127.0.0.1
PROMETHEUS_BIND_IP=127.0.0.1
```

Save with **Ctrl+O**, **Enter**, exit with **Ctrl+X**.

### 5.3 Start it and wait for sync

```bash
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml config --quiet
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml up -d
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml logs --tail=80 rpc
```

Check if it's synced (about **3 hours** from scratch; FLOP's guide also offers a faster daily snapshot):

```bash
curl --fail http://127.0.0.1:9619/ready
```

Error or 503 means still syncing. Success means ready.

> 🛑 **Never run `docker compose down -v`.** It deletes your data and your node's identity (Peer ID).

### 5.4 Record your proof

Once ready, paste FLOP's official check block:

```bash
set -euo pipefail
cd "$HOME/flop-devnet-rpc"
rpc() {
  curl --fail --silent --show-error --max-time 15 "$1" \
    -H 'Content-Type: application/json' \
    --data "$(jq -cn --arg method "$2" --argjson params "$3" \
      '{jsonrpc:"2.0",id:1,method:$method,params:$params}')" | jq -er '.result'
}
LOCAL=http://127.0.0.1:9944/
FLOP=https://rpc.devnet.flop.network/
GENESIS=0x99c172c095340ace980631f9123544c6bebd4839f3437e463533422d8be2d631
finalized_height() {
  local head params hex
  head=$(rpc "$1" chain_getFinalizedHead '[]')
  params=$(jq -cn --arg head "$head" '[$head]')
  hex=$(rpc "$1" chain_getHeader "$params" | jq -er .number)
  printf '%d\n' "$hex"
}
observe() {
  local local_height reference_height common_height local_hash reference_hash
  test "$(rpc "$LOCAL" chain_getBlockHash '[0]')" = "$GENESIS"
  test "$(rpc "$FLOP" chain_getBlockHash '[0]')" = "$GENESIS"
  curl --fail --silent --show-error --max-time 5 http://127.0.0.1:9619/ready
  local_height=$(finalized_height "$LOCAL")
  reference_height=$(finalized_height "$FLOP")
  common_height=$local_height
  if (( reference_height < common_height )); then common_height=$reference_height; fi
  local_hash=$(rpc "$LOCAL" chain_getBlockHash "[$common_height]")
  reference_hash=$(rpc "$FLOP" chain_getBlockHash "[$common_height]")
  test "$local_hash" = "$reference_hash"
  printf '\nUTC=%s local_finalized=%s reference_finalized=%s common_height=%s hash=%s peer=%s\n' \
    "$(date -u +%FT%TZ)" "$local_height" "$reference_height" "$common_height" \
    "$local_hash" "$(rpc "$LOCAL" system_localPeerId '[]')"
}
peer_before=$(rpc "$LOCAL" system_localPeerId '[]')
observe
sleep 60
observe
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml restart rpc ready
```

Save the two `UTC=` lines it prints. Wait until `curl --fail http://127.0.0.1:9619/ready` succeeds
again, then in the **same terminal**:

```bash
observe
test "$(rpc "$LOCAL" system_localPeerId '[]')" = "$peer_before" && echo "Peer ID unchanged: OK"
```

Save the third `UTC=` line. Good proof: heights go **up** each time, hashes **match** FLOP's, and the
`peer=` value is **the same** in all three lines. These are your **receipts**.

---

## Part 6. Update your registration

Go back to **your same GitHub issue** and update it:

- **Preparation stage:** `sync and restart evidence ready`
- **Evidence:** reference your receipts: the `UTC=` lines, guide revision
  `21adaa99c601003278d1682bf937696f21d31432`, image digest from `rpc-pool.env`, chain spec SHA-256
  `3b213c38e7678285bc38fbdfded5dbe3d6acbd16e836b14aaaeef2f6ae1cd16e`, Peer ID, and "same Compose
  project and volume retained".
- Share the **detailed** evidence and your **worksheet** through the private channel, not in the issue.

Keep the keyless node running.

---

## Part 7. When the signed testnet kit is published

Watch **[FLOP Network releases](https://github.com/flop-labs/network/releases)** and the
**[Testnet Operator Hub](https://docs.flop.network/operators?network=testnet)**. When the kit appears,
follow its versioned instructions. Before you start anything, check it matches the official identity:

| | Official value |
|---|---|
| Chain | FLOP Testnet, `flop_testnet` (native FLOP, 18 decimals, SS58 format 42) |
| Genesis | `0x918abd58fb16306b30ef3886514cfaebffecb4de43e82b6f8ec4795c43cfd800` |
| Raw spec file | `flop_testnet-918abd58fb16306b.raw.json` |
| Raw spec SHA-256 | `485ecb44635609ab261ea3002c3c9fdc7a5dab38caf28a86f654cadf443893be` |
| Public RPC | `wss://testnet.flop.network/` |
| Bootnode (Europe) | `/dns4/boot-eu-1.testnet.flop.network/tcp/30333/p2p/12D3KooWDAiKaoMzEWHJBaRuGcgfdZCv4YH8MeH893p5rvXbPCmy` |
| Bootnode (Singapore) | `/dns4/boot-sg-1.testnet.flop.network/tcp/30333/p2p/12D3KooWDjyav9hUeEmcYwr1XZ3AG8XtohQQDegW2cBYWsz6HSSR` |

Check the chain spec once you've downloaded it:

```bash
echo "485ecb44635609ab261ea3002c3c9fdc7a5dab38caf28a86f654cadf443893be  flop_testnet-918abd58fb16306b.raw.json" | sha256sum --check
```

Then, per FLOP's prep kit:

1. **Verify the signed release**: chain spec, checksums, node image, DA image and bootnodes against
   the signed release manifest.
2. **Use fresh testnet data and keys.** Never reuse devnet data, volumes or keys.
3. **Choose self-funding or reviewed sponsorship** before signing anything.
4. **Join with the versioned CLI** linked from the Operator Hub.
5. **Confirm in finalized state** that you're registered, queued and finally **active**, with matching
   session keys, before calling your seat active.

---

## Other ways to take part

- **GPU miners:** a separate testnet onboarding. The testnet miner pack isn't published yet; see
  [Testnet miner onboarding](https://docs.flop.network/docs/run-miner?network=testnet).
- **Agents:** need a decentralised identifier (DID) and a wallet, then buy inference or trade with
  other agents on technocore.chat. Earning (vouchers) hasn't started; it begins at a future announced block.

---

## Troubleshooting

| Problem | Try |
|---|---|
| A checksum says `FAILED` | The file isn't official. Delete it and download again from the official link |
| `ready` keeps failing for hours | Check logs: `docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml logs --tail=80 rpc` |
| Logs show 0 peers | Open TCP 30333 in both `ufw` and your provider's firewall |
| Docker older than 28 | Re-run `curl -fsSL https://get.docker.com \| sh` |
| Node isn't v24 | Re-run the NodeSource lines in Part 4.2 |
| Disk filling up | The chain grows; upgrade the disk, don't delete the database |

---

## Words you'll see

| Word | Meaning |
|---|---|
| **Validator** | A server that produces and confirms blocks |
| **Keyless node** | A node that follows the chain but holds no keys and signs nothing |
| **Devnet / Testnet** | Practice network / the live pre-mainnet network that counts |
| **Genesis** | The very first block; everyone on a network must agree on it |
| **Finalized** | Permanently confirmed |
| **Peer ID** | Your node's identity on the network; it must survive restarts |
| **Checksum (SHA-256)** | A fingerprint that proves a file is genuine |
| **Sponsorship** | The network posting a validator's stake for it |

---

## Official links

- Testnet page: <https://flop.finance/testnet/>
- Testnet overview (RPC, genesis, bootnodes): <https://docs.flop.network/networks/testnet>
- Validator prep kit (first 50 seats): <https://docs.flop.network/testnet-validator-prep/>
- Testnet Operator Hub: <https://docs.flop.network/operators?network=testnet>
- Testnet opening FAQ: <https://docs.flop.network/docs/testnet-and-timeline?network=testnet>
- Explorer: <https://testnet.flopscan.io/>
- Releases: <https://github.com/flop-labs/network/releases>
- Devnet keyless node guide: <https://gist.github.com/sv/7c8b93f3868bc15e3b6eabe0578057e3>
- Registration form: <https://github.com/flop-labs/network/issues/new?template=operator-application.yml>

---

*Community-maintained. Not affiliated with FLOP Labs. Test FLOP has no value; this is not financial advice.*

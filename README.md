# FLOP Labs Testnet: A Beginner's Guide to Getting Ready

Never run a node before? This guide takes you from zero to a **submitted testnet validator
application with proof that your server works**, step by step, with every command to copy and paste.

Current as of 25 Nov 2026.

> **Unofficial community guide.** Every command here comes from FLOP Labs' own published material
> (linked at each step). Cross-check [docs.flop.network](https://docs.flop.network) and the
> [flop-labs/network](https://github.com/flop-labs/network) repo. Nothing here guarantees a seat.

---

## Part 0. Understand what you're doing (5 minutes)

**FLOP Labs** is building a blockchain for verified AI inference. Two kinds of machines run it:

- **Miners** run AI models on GPUs.
- **Validators** keep the chain running: they agree on blocks and store data. A validator is a normal
  **CPU server**. No GPU needed. This guide is about validators.

There are two networks:

| | Devnet | Testnet |
|---|---|---|
| What it is | A practice network, live now | The real dress rehearsal before mainnet |
| Does it count? | **No.** No rewards, no airdrop weight | **Yes.** It counts toward the genesis allocation |
| Can outsiders join as validators? | Only by invitation | **Not yet.** FLOP hasn't published the testnet kit |

**So how do you get ready for testnet today?** You prove you can run a node properly *on devnet*,
using a **keyless node**: a node that follows the chain but holds no keys and signs nothing. It's
free, it needs no permission, and it's exactly the proof FLOP's testnet application form asks for.

**The plan:**

1. Rent a server
2. Connect to it and install the basics
3. Download FLOP's files and check they're genuine
4. Start a keyless node and let it sync
5. Collect proof that it works (and survives a restart)
6. Submit your testnet application with that proof
7. Prepare for the validator handoff (when FLOP opens signing)

Time: about 1 hour of your work, plus about 3 hours of waiting for the sync.

> ⚠️ **$FLOP is not live.** Nobody can sell you tokens, testnet seats or "guaranteed" spots.
> FLOP will never ask for your seed phrase, private keys or access to your server.

---

## Part 1. Rent a server

Rent a **Linux VPS or dedicated server** from any hosting provider. Choose:

| Setting | Choose |
|---|---|
| Operating system | **Ubuntu 24.04 LTS** |
| CPU type | **x86-64** (also called amd64; not ARM) |
| Size for this guide | At least **4 vCPU, 16 GB RAM, 300 GB SSD** (FLOP's pilot reference) |
| Network | A **public IPv4 address** that doesn't change |

Plan ahead: a real testnet *validator* is expected to need about **8 dedicated cores, 32 GB RAM,
4 TB NVMe** and reliable 1 Gbps (FLOP's planning estimate). You can start smaller for this keyless
step and upgrade later.

Use a server that runs **only this**. Don't put it on a machine with wallets or other projects.

**In your provider's firewall / security settings**, allow inbound:

- **TCP 22** (SSH, so you can log in)
- **TCP 30333** (the FLOP node talks to other nodes on this port)

Write down your server's **public IP address** and **root password** (or SSH key) from the provider.

---

## Part 2. Connect to your server and install the basics

### 2.1 Log in

- **Windows:** open **PowerShell**. **Mac / Linux:** open **Terminal**.
- Type this, using your server's IP:

```bash
ssh root@YOUR_SERVER_IP
```

Type `yes` if asked about the fingerprint, then your password. You're in when the prompt shows your
server's name. Every command from now on runs **on the server**.

### 2.2 Update the system and install tools

```bash
apt update && apt upgrade -y
apt install -y curl jq ufw zstd
```

### 2.3 Install Docker

Docker runs the FLOP node in a container. This is Docker's official installer:

```bash
curl -fsSL https://get.docker.com | sh
docker --version && docker compose version
```

Both commands should print a version number.

### 2.4 Turn on the server's own firewall

```bash
ufw allow OpenSSH
ufw allow 30333/tcp
ufw --force enable
ufw status
```

You should see `22/tcp` (OpenSSH) and `30333/tcp` as **ALLOW**. Nothing else needs to be open.

---

## Part 3. Download FLOP's files and check they're genuine

These commands come straight from FLOP's official
[Devnet RPC guide](https://gist.github.com/sv/7c8b93f3868bc15e3b6eabe0578057e3#run-a-flop-devnet-rpc-replica).
They download five setup files pinned to an exact version, plus the chain's "genesis file", and check
every file's fingerprint (SHA-256). **If any line doesn't say `OK`, stop.** The file isn't genuine.

Copy and paste the whole block:

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

You should see six lines ending in **`OK`**.

---

## Part 4. Configure and start your keyless node

### 4.1 Find your two IP addresses

```bash
hostname -I | awk '{print $1}'     # your server's LOCAL IP  -> use for HOST_IP
curl -4 -s https://api.ipify.org; echo   # your PUBLIC IP -> use in PUBLIC_ADDR
```

On many VPS providers both are the same. On AWS-style cloud servers they're different; that's fine.

### 4.2 Edit the settings file

```bash
nano rpc-pool.env
```

Find these two lines and fill them in (keep everything else as it is):

```text
HOST_IP=YOUR_LOCAL_IP
PUBLIC_ADDR=/ip4/YOUR_PUBLIC_IP/tcp/30333
```

Make sure these stay exactly like this, so your node's private ports are not exposed:

```text
RPC_BIND_IP=127.0.0.1
PROMETHEUS_BIND_IP=127.0.0.1
```

Save with **Ctrl+O**, **Enter**, then exit with **Ctrl+X**.

The file already pins FLOP's official node image and bootnode. Don't change those.

### 4.3 Start the node

```bash
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml config --quiet
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml up -d
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml logs --tail=80 rpc
```

The first command prints nothing if your settings are valid. The last one shows the node's log:
you should see it finding peers and importing blocks.

### 4.4 Wait for it to sync

Check if it's ready:

```bash
curl --fail http://127.0.0.1:9619/ready
```

- **Error / 503** means it's still catching up. That's normal. A full sync takes about **3 hours**.
- **Success (200)** means it's synced and following the chain.

Check again every half hour or so. Want it faster? FLOP publishes a daily database snapshot you can
start from; see "Optional: start from a FLOP snapshot" in the
[official guide](https://gist.github.com/sv/7c8b93f3868bc15e3b6eabe0578057e3#run-a-flop-devnet-rpc-replica).

> 🛑 **Never run `docker compose down -v`.** The `-v` deletes your node's data *and its identity*
> (Peer ID), and you'd have to start over.

---

## Part 5. Collect your proof

Once the readiness check succeeds, run FLOP's official checks. They confirm your node agrees with
FLOP's own node about the chain, take two snapshots a minute apart, then restart your node.

Paste this whole block (from the official guide):

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

You'll see **two lines** starting with `UTC=`. **Copy them somewhere safe**: that's Observation 1
and Observation 2.

Now wait a few minutes for the node to come back (`curl --fail http://127.0.0.1:9619/ready` succeeds
again), then in the **same terminal window** run:

```bash
observe
test "$(rpc "$LOCAL" system_localPeerId '[]')" = "$peer_before" && echo "Peer ID unchanged: OK"
```

Copy the third `UTC=` line (that's your **after restart** observation) and check you see
`Peer ID unchanged: OK`.

**What good proof looks like:** the finalized heights go up each time, the hashes match FLOP's, and the
`peer=` value is the same in all three lines. If anything fails, don't delete your node; note the
error and mention it in your application.

> All of this is **public** information. It contains no keys or secrets.

---

## Part 6. Submit your testnet application

1. You need a free **GitHub account**.
2. Open the
   [Validator / RPC operator application](https://github.com/flop-labs/network/issues/new?template=operator-application.yml)
   in `flop-labs/network` (or pick it from the
   [issue chooser](https://github.com/flop-labs/network/issues/new/choose)).
3. Fill it in:

| Field | What to write |
|---|---|
| Team ID | A short public name you'll keep using (e.g. your handle) |
| Network | **testnet** |
| Requested role | **validator candidate** |
| Existing application | Leave empty if this is your first one |
| Planned capacity and host | `1 seat`, your provider and region, CPU / RAM / disk / OS, and your public P2P address (`/ip4/YOUR_PUBLIC_IP/tcp/30333`) |
| Preparation stage | **sync and restart evidence ready** |
| Sync and restart evidence | Paste your 3 `UTC=` lines, plus: guide revision `21adaa99c601003278d1682bf937696f21d31432`, chain spec SHA-256 `3b213c38…1cd16e`, expected genesis `0x99c172c0…2d631`, "same Compose project and volume retained, no Tailscale on this host", and your Peer ID |

4. Submit. **Your issue number is your application ID.**

Rules that matter:

- **One application per team per network.** To add or change something, edit or comment on your
  existing issue. Never open a second one.
- The issue is **public**: never paste seeds, private keys, `account.json`, `.env`, keystores or
  tokens.
- Submitting doesn't grant a seat. FLOP verifies your evidence and labels your issue as you progress.

---

## Part 7. From keyless node to testnet validator

Your keyless node and application are **stage 1**. Becoming a signing validator follows FLOP's
official handoff sequence for new public validators
([signing release and handoff](https://gist.github.com/sv/870b0fa9d63efa6c7873ded10d49d411)).
**Signing is not open to the public yet**: FLOP activates new validators only after it publishes a
qualified signing package and network inputs. Here's the path, and what you can prepare **now**.

### The official handoff sequence

1. **Keep your keyless node running.** Same Peer ID, same volume. RPC, readiness and metrics stay
   on localhost.
2. **Receive the signing package and network inputs from FLOP.** A hardened node image, a
   checksummed chain spec, public peers, approved public DA routes and capacity requirements.
   **Until you have this, don't try to set up signing.**
3. **Prepare locally.** Your private keys stay on your own server. You only ever share your
   **public SS58 address** and your **public P2P and DA endpoints** (in your application issue).
   No Tailscale, no SSH keys, no copied keystores are requested.
4. **Pass the technical preflight.** Correct genesis, finality advancing in agreement with FLOP,
   P2P and DA reachable in both directions, enough storage, correct clock, and RPC/metrics closed to
   the internet.
5. **Register only at the handoff.** FLOP checks your account and stake or sponsorship eligibility,
   then you run the package's tested registration step yourself.
6. **Activation, one at a time.** FLOP activates a canary validator first, verifies it really authors
   and confirms blocks and handles its data duties, then expands the group.

### Prepare now (copy-paste)

**a) Plan a separate validator server.** Keep your keyless node as is. The validator should be its
own persistent server sized to FLOP's planning estimate: **8 dedicated cores, 32 GB RAM, 4 TB NVMe**,
reliable 1 Gbps, stable public IPv4.

**b) Open the validator's ports** (on the *validator* server; also open them in your provider's firewall):

```bash
ufw allow OpenSSH
ufw allow 30333/tcp   # P2P: other nodes connect to yours
ufw allow 7001/tcp    # data availability (DA): public data endpoint
ufw --force enable
```

RPC (9944) and metrics (9615) must **never** be opened to the internet.

**c) Install Cosign**, the tool used to check that a FLOP image is genuinely signed by FLOP:

```bash
curl -fsSL -o /usr/local/bin/cosign \
  https://github.com/sigstore/cosign/releases/download/v2.4.1/cosign-linux-amd64
chmod +x /usr/local/bin/cosign
cosign version
```

**d) Practice verifying an official image.** When FLOP publishes the testnet image, you'll verify
it exactly like this. Here's the same check on FLOP's published devnet image (it only downloads and
inspects; it doesn't run a validator):

```bash
IMAGE=ghcr.io/flop-labs/flop-core/node-devnet:v0.1.435@sha256:bc88eae43880413e1e9ce8363c0bb0d4d7f1de7643d00f21b59de0bde33c650f
cosign verify \
  --certificate-identity 'https://github.com/flop-labs/flop-core/.github/workflows/build-services.yml@refs/heads/main' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com "$IMAGE"
docker pull "$IMAGE"
docker run --rm --network none --entrypoint zkv-relay "$IMAGE" --version
```

`cosign verify` must print a successful verification. Always use the exact `@sha256:` digest FLOP
publishes, never `latest`.

**e) Add your validator details to your application.** Comment on your **same** issue with your
validator server's public IP, its planned P2P address (`/ip4/IP/tcp/30333`), your planned DA endpoint
(`http://IP:7001`) and the server specs. Never post private keys.

### What the testnet validator setup will look like

From the `public-testnet` profile already shipped in FLOP's validator package:

- Only **two public ports**: P2P **TCP 30333** and the filtered DA endpoint **TCP 7001**.
- **Numeric public IPv4** only in the first version (no DNS or IPv6).
- RPC forced to safe methods and bound to localhost; the node runs as an archive node.
- **Testnet has its own image, chain spec, genesis and bootnodes.** Never reuse devnet keys,
  volumes or settings for testnet.

---

## Part 8. While you wait

- **Keep the node running.** Check it now and then: `curl --fail http://127.0.0.1:9619/ready`.
- **Watch disk space:** `df -h /`. The chain keeps growing; upgrade before it fills up.
- **Watch for FLOP's testnet announcement** in the [flop-labs/network](https://github.com/flop-labs/network)
  repo and on [docs.flop.network](https://docs.flop.network). When the testnet kit is published:
  - Download it **only from official FLOP sources** and verify every checksum.
  - Use a **fresh** testnet setup. Never reuse your devnet database, volume or keys.
  - Follow the official admission steps and update your **same** application issue.

---

## Troubleshooting

| Problem | What it usually means | Try |
|---|---|---|
| `ready` keeps failing after many hours | Still syncing, or no peers | Check `docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml logs --tail=80 rpc` |
| Logs show 0 peers | Port 30333 is blocked | Open TCP 30333 in **both** your provider's firewall and `ufw` |
| A checksum says `FAILED` | The file isn't the official one | Delete the folder and start Part 3 again |
| Disk full | The archive outgrew your disk | Upgrade the server's disk; don't delete the database |
| `docker: command not found` | Docker isn't installed | Redo Part 2.3 |

---

## Words you'll see

| Word | Meaning |
|---|---|
| **Node** | A computer running the blockchain software |
| **Validator** | A node that helps produce and confirm blocks (needs admission) |
| **Keyless node** | A node that follows the chain but holds no keys and signs nothing |
| **Devnet / Testnet** | Practice network / the rehearsal network that counts |
| **Genesis** | The very first block. Every node on the same network must agree on it |
| **Finalized height** | How many blocks are permanently confirmed |
| **Peer ID** | Your node's identity on the network. It must stay the same across restarts |
| **Checksum (SHA-256)** | A fingerprint proving a file wasn't changed |
| **Port 30333** | The door other nodes use to reach yours |

---

## Official sources

- FLOP Devnet RPC guide (Parts 3 to 5): <https://gist.github.com/sv/7c8b93f3868bc15e3b6eabe0578057e3>
- FLOP signing release and validator handoff (Part 7): <https://gist.github.com/sv/870b0fa9d63efa6c7873ded10d49d411>
- Docs: <https://docs.flop.network> · search: <https://ask.flop.finance>
- Applications: <https://github.com/flop-labs/network>

---

*Community-maintained. Not affiliated with FLOP Labs. $FLOP is not live; this is not financial advice.*

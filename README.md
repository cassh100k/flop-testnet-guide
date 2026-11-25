# Getting Ready for the FLOP Labs Testnet

What you can do **today** to be first in line when the FLOP testnet opens to outside validators.
Current as of 25 Nov 2026.

> **Unofficial community guide.** Cross-check [docs.flop.network](https://docs.flop.network),
> [ask.flop.finance](https://ask.flop.finance) and the
> [flop-labs/network](https://github.com/flop-labs/network) repo. Nothing here grants a seat.

Running a devnet validator? See the **[devnet guide](https://github.com/cassh100k/flop-validator-guide)**.

---

## Where things stand

- **Testnet is the network that counts** toward the genesis allocation. Devnet does not.
- **You can apply now**, but an application *records interest*. Outside validators can't actually
  run a testnet validator yet: the testnet's node image, raw chain spec, genesis, bootnodes, DA profile
  and admission instructions have **not been published**.
- The validator package already contains a `public-testnet` profile, but every public value in it is
  deliberately blank, and it refuses to start until FLOP publishes a reviewed network identity and
  launch approvals. Don't try to fill those values in yourself.
- **Validators will be admitted via sponsorship** within on-chain limits (no allowlist for
  validators). Sponsorship alone does not mean activation.
- **$FLOP is not live.** Anyone selling testnet "seats", tokens or airdrops is a scammer.

So the winning move now is: **apply, prove your node can sync, and have the right hardware ready.**

---

## Step 1. Apply

Open an issue with the **"Validator / RPC operator application"** template in
[flop-labs/network](https://github.com/flop-labs/network/issues/new/choose).

| Field | What to put |
|---|---|
| Team ID | A stable public name you'll reuse for all your nodes and receipts |
| Network | `testnet` |
| Requested role | `validator candidate` (or `... and RPC contributor`) |
| Existing application | Link your devnet issue or [network#10](https://github.com/flop-labs/network/issues/10) comment, if any |
| Planned capacity and host | Seats requested, provider/region, CPU / RAM / SSD / OS, public P2P address if prepared |
| Preparation stage | Be honest: `planning a host` → `syncing a keyless node` → `sync and restart evidence ready` |
| Sync and restart evidence | Fill in after Step 2 (you can edit the issue later) |

Rules that matter:

- **One application per team per network.** Update your issue instead of opening another.
- The issue is **public**. Never include seeds, private keys, `account.json`, `.env`, keystores,
  enrollment tokens or TLS private keys. **No SSH access is requested.**
- Start with **one keyless node**. Don't rent a whole fleet before review.

## Step 2. Prove your node with a keyless devnet sync

This is the evidence the application asks for, and you can do it **today without Tailscale or any
admission**. Follow FLOP's official guide exactly:
**[Run a FLOP Devnet RPC replica](https://gist.github.com/sv/7c8b93f3868bc15e3b6eabe0578057e3#run-a-flop-devnet-rpc-replica)**.

In short:

1. Download the five pinned files from the guide (`docker-compose.rpc.yml`, `node-entrypoint.sh`,
   `rpc_ready.py`, `rpc-pool.env.example`, `rpc-origin.Caddyfile`) and check them with `sha256sum --check`.
2. Download the devnet raw spec and verify it:
   `flop_devnet_ts-99c172c095340ace.raw.json`, SHA-256
   `3b213c38e7678285bc38fbdfded5dbe3d6acbd16e836b14aaaeef2f6ae1cd16e`.
3. `cp -n rpc-pool.env.example rpc-pool.env`, then set the node image digest, chain spec, bootnode,
   your `HOST_IP` and `PUBLIC_ADDR` exactly as the guide says. Keep RPC (9944), readiness (9619) and
   metrics (9615) on `127.0.0.1`. Only TCP **30333** faces the internet.
4. Start it:
   ```sh
   docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml config --quiet
   docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml up -d
   ```
   A full sync from genesis takes roughly 3 hours (the guide also documents a verified snapshot).
5. Wait until `curl --fail http://127.0.0.1:9619/ready` succeeds.

**Collect the evidence** (all public values):

- Genesis on your node (`chain_getBlockHash [0]`) equals
  `0x99c172c095340ace980631f9123544c6bebd4839f3437e463533422d8be2d631`, same as
  `https://rpc.devnet.flop.network/`.
- **Observation 1:** UTC time, your finalized height, the reference's finalized height, the hash at
  the lower of the two heights (must match on both), readiness.
- Wait 60 seconds. **Observation 2:** same values.
- Restart: `docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml restart rpc ready`,
  wait for ready. **Observation 3:** same values, finality still advancing.
- Your **Peer ID** (`system_localPeerId`) is **unchanged** after the restart, and you kept the same
  Compose project and volume.
- The exact image digest and chain-spec checksum you ran.

Paste those into your application and set the stage to `sync and restart evidence ready`.

> Never `docker compose down -v` this node. The volume holds `node_key.dat`, which is your Peer ID.

## Step 3. Have the right host

| | Keyless devnet sync (Step 2) | Testnet validator (planning) |
|---|---|---|
| CPU | 4 vCPU (pilot reference) | 8 dedicated cores |
| RAM | 16 GiB | 32 GB |
| Disk | 300 GB SSD, archive keeps growing | 4 TB NVMe incl. DA custody |
| Network | Stable public IPv4 | Reliable 1 Gbps, stable public IPv4 |

All figures are FLOP's published references and planning estimates, not guaranteed minimums.
Validators are judged on uptime: use a **dedicated** machine, not a shared box.

## Step 4. Know what the testnet setup will look like

From the `public-testnet` profile already in the validator package:

- **Only two public ports:** P2P **TCP 30333** and the filtered DA ingress **TCP 7001**. RPC (9944) and
  metrics (9615) are loopback-only, RPC is forced to `safe` methods.
- **Numeric public IPv4 only** for now (no DNS or IPv6 in the first profile).
- The node runs as an **archive** node.
- It's a standalone Compose file. **Never layer it over the devnet Compose file.**
- Testnet will have **its own** image, chain spec, genesis and bootnodes. **Do not reuse devnet
  genesis, volumes, keys or enrollment material** for testnet.

## Step 5. When FLOP publishes the testnet kit

1. Download it only from official FLOP sources and **verify every checksum and signature**.
2. Check the published genesis on at least two independent sources before signing anything.
3. Generate **fresh** testnet keys on the dedicated host. Never move or reuse devnet keys.
4. Follow the official sponsorship/admission steps, and update your application issue.

---

## Avoid scams

- FLOP will **never** ask for your seed, private keys or SSH access in an issue, DM or form.
- There is no paid "fast-track" to a validator seat, and nothing to buy.
- Only download packages from FLOP's official pages, and always verify the checksum.

---

*Community-maintained. Not affiliated with FLOP Labs. $FLOP is not live; this is not financial advice.*

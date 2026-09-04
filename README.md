# Pixagram witness

Minimal docker-compose stack for a Pixagram witness node. It joins the network
over P2P at `api.pixagram.com:2001`, validates blocks and produces blocks in
your scheduled slot. Nothing else: no HAF, no PostgreSQL, no Hivemind, and no
usable API.

Only the `witness` plugin is loaded. hived still starts a webserver, because it
comes in as a transitive dependency, but no API plugins are registered so every
method returns `Could not find API <name>`. Query the chain against
`https://api.pixagram.com` instead.

If you want the social API (`bridge.*`, follow, tags), you want an API node
instead — that is a different, much heavier stack.

---

## ⚠️ Required edits before first start

**1. `pixagram/config.ini`** — without these two lines the node syncs happily
and never signs anything:

```ini
witness     = "your_witness_account"        # ← your registered witness account
private-key = 5xxxxxxxxxxxxxxxxxxxxxx...    # ← your witness SIGNING key (WIF)
```

| What | What to put there |
|---|---|
| `witness` | Your witness account name, as registered on chain with `witness_update_operation`. It must be set on-chain **before** this node can be scheduled. |
| `private-key` | The signing key (WIF, starts with `5`) matching the `block_signing_key` on your witness object. **Not** your owner key and **not** your active key. |

**Treat `config.ini` as secret.** That WIF lets anyone produce blocks as your
witness — or miss them, which gets your witness disabled. `.gitignore` covers
`.env` and the datadir but **not** `config.ini`, because the file is tracked as
a template. Once you fill in your key, `chmod 600` it and never commit it.

---

## Sizing

Measured on a production Pixagram node, except where marked.

| | Full API node (measured) | Witness only |
|---|---:|---:|
| RAM | 5.4 GB | ~1.0–1.5 GB *(estimate)* |
| CPU, steady state | 0.21 vCPU | below that |
| Disk, chain data | 5.1 GB | ~0.4 GB |
| Containers | 9 | 2 |

**Recommended: 2 vCPU / 4 GB / 50 GB SSD.** 4 vCPU / 8 GB / 100 GB gives
comfortable headroom for chain growth and costs little more.

The witness RAM figure is derived rather than measured: a full consensus node
measured 2 224 MB with `account_history_rocksdb` enabled, and that plugin's
rocksdb cache is the bulk of what disappears when it is off. Treat ~1.5 GB as
the working assumption.

Nearly all of the saving comes from one plugin. `account_history_rocksdb`
accounted for **1.2 GB of a 1.57 GB datadir** — 76% of the disk — plus a large
in-memory cache. A witness answers no history queries.

---

## Start

```bash
docker compose up -d
docker compose logs -f pixagram
```

The node syncs from the seed over P2P, then follows live blocks. Once it is
caught up and your witness is in the active schedule you will see:

```
witness_plugin.cpp:423   Generated block #N with timestamp ...
```

If instead you see:

```
witness_plugin.cpp:338   Won't produce block because I don't have the private key for PIX...
```

your `private-key` does not match your witness's on-chain `block_signing_key`.
Fix it and restart.

---

## Ports

| Port | Exposure | Required |
|---|---|---|
| `2001/tcp` | **open to `0.0.0.0/0`** | **yes** — no P2P, no sync, no blocks |

That is the only published port. hived's webserver listens inside the container
but is not exposed, and would answer nothing useful anyway.

---

## Two settings you must NOT copy from the bootstrap node

```ini
enable-stale-production = true
required-participation  = 0
```

The node that starts the chain carries both, because it has to produce blocks
alone with no peers. On a node joining an existing network they remove exactly
the protections you want: the first makes you build on a chain you already know
is stale, the second drops the check that stops you producing during a network
split. Leave both unset, as they are in the shipped config.

---

## Verify it is actually working

This node exposes no API, so check it from the outside and from its logs.

```bash
# your witness object - signing key and, more importantly, missed count
curl -s -X POST https://api.pixagram.com -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"condenser_api.get_witness_by_account","params":["your-account"],"id":1}'

# is your node keeping up?
docker compose logs --tail=50 pixagram | grep -E "Generated block|Syncing|entering live mode"
```

`total_missed` is the number that matters. If it climbs, your node is not
producing when scheduled.

---

## Files

| Path | Purpose |
|---|---|
| `docker-compose.yml` | `pixagram` (hived, `pixadock/pixagram:mainnet`) |
| `pixagram/config.ini` | hived config — consensus plugins only |
| `pixagram/` | mounted as the datadir; holds `blockchain/`, `p2p/` after first run |

## Operating

```bash
docker compose ps                    # status
docker compose logs -f pixagram      # follow logs
docker compose restart pixagram      # after a config edit
docker compose down                  # stop
```

## Resetting / resyncing from genesis

```bash
docker compose down
sudo rm -rf pixagram/blockchain pixagram/p2p pixagram/logs pixagram/*.log
docker compose up -d
```

`config.ini` is preserved.

`block_log*` is the chain itself — never delete it on a node you intend to keep.
Everything else under `blockchain/` is derived state and is rebuilt by a replay:

```bash
docker compose down
rm -f pixagram/blockchain/shared_memory.bin
HIVED_EXTRA_ARGS=--replay-blockchain docker compose up -d pixagram
# once it logs "entering live mode":
docker compose up -d
```

## Price feed

Witnesses are expected to publish a price feed, and a stale one counts against
you. This repo does not ship a feed publisher, because a node running only the
`witness` plugin has no `network_broadcast_api` to broadcast through.

Publish it from somewhere that does have an API — sign locally with your
account's **active** key (not the signing key in `config.ini`) and broadcast via
`https://api.pixagram.com` or your own API node.

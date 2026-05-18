# Pixagram witness

Minimal docker-compose stack to run a Pixagram witness node. Connects to the public pre-mainnet at `api.pixagram.com:2001` over P2P, produces blocks when scheduled, and does nothing else — no RPC, no APIs, no Hivemind, no HAF.

---

## ⚠️ Required edits before first start

You **must** edit `pixagram/config.ini` and fill in these two lines, otherwise the node will run but never produce a block:

```ini
# pixagram/config.ini

witness     = "your_witness_account"        # ← REPLACE: your registered witness account name
private-key = 5xxxxxxxxxxxxxxxxxxxxxx...    # ← REPLACE: the WIF of your witness signing key
```

| What | What to put there |
|---|---|
| `witness` | The account name of your witness as registered on chain via `witness_update_operation`. Must be set on-chain BEFORE this node will be scheduled. |
| `private-key` | The signing key (WIF, starts with `5...`) that matches the `block_signing_key` you set on your witness object. NOT your account owner/active key. |

If either is missing or wrong, hived will start, sync the chain, and just sit there — it won't sign any blocks.

**Treat `config.ini` as a secret file** — the WIF gives anyone the ability to produce blocks as your witness (or miss them, getting your witness disabled). The `.gitignore` excludes datadir contents but **does not** exclude `config.ini`; if you commit, scrub the key first or use a `config.ini.local` symlink (the `.gitignore` already ignores `*.local`).

---

## Start

```bash
docker compose up -d
docker compose logs -f pixagram      # watch it sync + produce
```

The node will sync the chain from genesis over P2P, then follow live blocks. Once it's caught up and your witness is in the active schedule, you'll see lines like:

```
witness_plugin.cpp:423   Generated block #N with timestamp ...
```

If you see:

```
witness_plugin.cpp:338   Won't produce block because I don't have the private key for PIX...
```

then your `private-key` doesn't match your witness's on-chain `block_signing_key` — fix it and restart.

## Ports

| Port | Purpose |
|---|---|
| `2001` | P2P (open to internet so peers can reach you; also outbound to `api.pixagram.com:2001`) |

No HTTP, no WS — there's no RPC server in this image. If you need to query the chain, do it against `https://api.pixagram.com` instead.

## Files

| Path | Purpose |
|---|---|
| `docker-compose.yml` | Single `pixagram` service running `pixadock/pixagram:pre-mainnet` |
| `pixagram/config.ini` | hived config — only the `witness` plugin loaded |
| `pixagram/` | Mounted as the datadir; will hold `blockchain/`, `p2p/`, etc. after first run |

## Operating

```bash
docker compose ps                    # status
docker compose logs -f pixagram      # follow logs
docker compose restart pixagram      # restart after config edit (e.g. key change)
docker compose down                  # stop
```

## Resetting / resyncing from genesis

If something goes wrong or you switch chains:

```bash
docker compose down
sudo rm -rf pixagram/blockchain pixagram/p2p pixagram/logs pixagram/*.log
docker compose up -d
```

Your `config.ini` is preserved.

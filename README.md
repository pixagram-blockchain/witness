# Pixagram witness

Minimal docker-compose stack for a Pixagram witness node. It joins the network
over P2P at `api.pixagram.com:2001`, validates blocks and produces blocks in
your scheduled slot, and publishes a price feed. No HAF, no PostgreSQL, no
Hivemind, no public API.

Four plugins are loaded: `witness` to produce blocks, plus `database_api`,
`condenser_api` and `network_broadcast_api` so the price feed can broadcast
**through this node** rather than through somebody else's. hived's HTTP endpoint listens only inside
the compose network — it is not published to the host or the internet.

For general chain queries use `https://api.pixagram.com`; this node is not an
API node.

If you want the social API (`bridge.*`, follow, tags), you want an API node
instead — that is a different, much heavier stack.

---

## ⚠️ Required edits before first start

**1. `pixagram/config.ini`** — uncomment these two lines and fill them in:

```ini
witness     = "your_witness_account"        # ← your registered witness account
private-key = 5xxxxxxxxxxxxxxxxxxxxxx...    # ← your witness SIGNING key (WIF)
```

They ship commented out deliberately. hived parses `private-key` while the
witness plugin initialises and aborts outright if the option is present but
empty — `unable to parse private key` — so leaving a bare `private-key =` in
the file gives you a crash loop rather than a node that simply does not
produce.

| What | What to put there |
|---|---|
| `witness` | Your witness account name, as registered on chain with `witness_update_operation`. It must be set on-chain **before** this node can be scheduled. |
| `private-key` | The signing key (WIF, starts with `5`) matching the `block_signing_key` on your witness object. **Not** your owner key and **not** your active key. |

**2. `.env`** — for the price feed:

```bash
cat > .env <<'EOF'
WITNESS_ACCOUNT=your-account
WITNESS_WIF=your-ACTIVE-key
EOF
chmod 600 .env
```

The two keys are different, and mixing them up is a common mistake.
`config.ini` takes your **signing** key, which signs blocks. `.env` takes your
account's **active** key, which signs the `feed_publish` operation.

**Treat `config.ini` as secret.** That WIF lets anyone produce blocks as your
witness — or miss them, which gets your witness disabled. `.gitignore` covers
`.env` and the datadir but **not** `config.ini`, because the file is tracked as
a template. Once you fill in your key, `chmod 600` it and never commit it.

**Then protect it from `git pull`.** Because the file is tracked, a later
`git pull` — or any `git reset --hard` — replaces your filled-in copy with the
placeholder version. This failure is quiet and delayed: hived reads `config.ini`
once at startup and never again, so the running node keeps producing normally
and nothing looks wrong. It stops producing the next time the container
restarts, which may be days later. Tell git to leave your copy alone:

```bash
chmod 600 pixagram/config.ini
git update-index --skip-worktree pixagram/config.ini
```

`git ls-files -v pixagram/config.ini` should then print a leading `S`. To pick
up a genuine upstream change to the template later:

```bash
git update-index --no-skip-worktree pixagram/config.ini
cp pixagram/config.ini pixagram/config.ini.mine    # your identity, kept aside
git stash && git pull && git stash pop             # resolve any conflict
git update-index --skip-worktree pixagram/config.ini
```

---

## ⚠️ Required: a synchronised clock

Your machine must run an NTP daemon. This is not a nicety: a witness with a
drifting clock silently costs **other** witnesses their blocks.

Block slots are 3 seconds, and hived decides when to produce from the local
system clock. If your clock is a second or two slow, your block goes out that
late in real time and reaches your peers at the boundary of the *next* witness's
slot. That witness has not seen your block, so it builds on the previous head,
produces a competing block at the same height, and loses the fork race. It is
recorded as a missed block **for them**.

That is the part worth understanding: **the penalty lands on the next witness in
the schedule, not on you.** Your own `total_missed` stays clean while you cost
the network blocks, so you will never notice it from your own numbers.

This is not hypothetical. One Pixagram witness ran 2.672 s slow and was
responsible for 121 of the 123 same-height block collisions on the network; a
single witness on the far side of the world absorbed 109 missed blocks because
of it. Installing an NTP daemon brought the offset to 3 ms and the misses
stopped immediately — the next 3.5 hours produced 4163 blocks from 4163 slots,
with not one empty.

**Check what you have:**

```bash
timedatectl | grep -E 'synchronized|NTP service'
```

You want `System clock synchronized: yes` and an active NTP service. Ubuntu
ships `systemd-timesyncd` enabled and most cloud images do the right thing, but
**Debian cloud images frequently ship with no time daemon at all** — which is
exactly how the incident above happened.

**If it is missing:**

```bash
sudo apt-get update && sudo apt-get install -y chrony
sudo systemctl enable --now chrony
```

Make sure it is `enabled`, not merely started, or it will not survive a reboot.

**Verify the real offset.** Do not trust `timedatectl` alone, and do not compare
`date` over SSH from your laptop — connection latency swamps the number you are
trying to measure. Ask an NTP server from the machine itself:

```bash
python3 - <<'PY'
import socket, struct, time, statistics
def probe(host):
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM); s.settimeout(4)
    try:
        t1 = time.time(); s.sendto(b'\x1b' + 47 * b'\0', (host, 123))
        d, _ = s.recvfrom(1024); t4 = time.time()
    finally:
        s.close()
    u = struct.unpack('!12I', d[:48])
    t2 = u[8] + u[9] / 2**32 - 2208988800
    t3 = u[10] + u[11] / 2**32 - 2208988800
    return ((t2 - t1) + (t3 - t4)) / 2
offsets = []
for host in ('pool.ntp.org', 'time.google.com', 'time.cloudflare.com'):
    for _ in range(3):
        try: offsets.append(probe(host))
        except Exception: pass
print(f'clock offset: {statistics.median(offsets):+.4f}s  ({len(offsets)} samples)'
      if offsets else 'could not reach any NTP server')
PY
```

Under ~50 ms is fine. Past roughly half a second you are costing your neighbours
in the schedule real blocks.

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
| `7777/tcp` | compose network only | internal; **not** published to the host |

2001 is the only published port. 7777 carries the price feed's traffic to hived
over the internal docker network and has no host binding at all, so it is
unreachable from outside the machine. Do not add one: a witness serving a
public API attracts load that makes it miss blocks.

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

# would it still produce after a restart? both lines must be present and uncommented
grep -E '^(witness|private-key) =' pixagram/config.ini
```

`total_missed` is the number that matters. If it climbs, your node is not
producing when scheduled.

---

## Files

| Path | Purpose |
|---|---|
| `docker-compose.yml` | `pixagram` (hived, `pixadock/pixagram:mainnet`) |
| `pixagram/config.ini` | hived config — witness plus the two APIs the feed needs |
| `.env` | `WITNESS_ACCOUNT` and `WITNESS_WIF` for the feed (git-ignored) |
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

`config.ini` and `.env` are preserved.

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

The `bigmac-feed` container publishes a feed where 1 PXS tracks the price of a
Big Mac. Witnesses are expected to publish one, and a stale feed counts against
you.

It broadcasts through your own node over the compose network
(`--rpc=http://pixagram:7777`), which is the reason `database_api` and
`network_broadcast_api` are loaded. Your signed transaction never passes
through a third party.

```bash
docker compose logs -f bigmac-feed
```

Adjust `--token-price` in `docker-compose.yml` if the PIXA/USD reference moves.
Remove the service if you publish your feed by other means.

# Restic Backup to S3-Compatible Storage

A practical guide to setting up encrypted, deduplicated backups with restic and an S3-compatible storage provider. Covers two real-world use cases: PostgreSQL database backup via stdin pipe, and a Docker volume directory backup.

---

## Architecture Overview

```
VPS — PostgreSQL (Docker)
└── cron → backup.sh (host)
      ├── docker exec postgres-db pg_dump
      │         |
      │        pipe (on host)
      │         |
      └── docker run restic/restic backup --stdin → S3

Home Server — Vaultwarden (VM)
└── cron → backup.sh (host)
      └── docker run restic/restic backup /data → S3
```

**Key design decisions:**
- Restic runs as a one-shot Docker container (`--rm`), no persistent daemon
- All orchestration happens on the host via cron + bash script
- Each repository uses a separate encryption password
- DNS is explicitly set to `1.1.1.1` to bypass VPN/mesh DNS that may not resolve external domains inside Docker containers

---

## Storage

| Parameter | Value |
|---|---|
| Provider | S3-compatible (e.g. Infomaniak Swiss Backup, Backblaze B2, AWS S3) |
| Endpoint | `s3.your-provider.com` |
| Repo — VPS | `s3:s3.your-provider.com/YOUR_BUCKET/vps-postgres` |
| Repo — Home | `s3:s3.your-provider.com/YOUR_BUCKET/home-vaultwarden` |

Each repository is encrypted with a **separate password**. If using a single set of S3 credentials for both servers, repositories are still isolated at the encryption level — a compromised server can't read the other repo's data.

---

## VPS — PostgreSQL Backup

### File structure

```
/opt/restic/
├── .env
└── backup.sh
```

### `.env`

```bash
AWS_ACCESS_KEY_ID=YOUR_KEY
AWS_SECRET_ACCESS_KEY=YOUR_SECRET
RESTIC_REPOSITORY=s3:s3.your-provider.com/YOUR_BUCKET/vps-postgres
RESTIC_PASSWORD=YOUR_STRONG_PASSWORD
TG_BOT=YOUR_BOT_TOKEN
TG_CHAT=YOUR_CHAT_ID
```

```bash
chmod 600 /opt/restic/.env
```

### `backup.sh`

```bash
#!/bin/bash
set -euo pipefail

source /opt/restic/.env

echo "[$(date)] Starting backup..."

# pg_dump pipes directly into restic — no temp file on disk
docker exec postgres-db pg_dump -U postgres postgres \
  | docker run --rm -i \
      --dns 1.1.1.1 \
      --env-file /opt/restic/.env \
      -v restic-cache:/cache \
      restic/restic backup \
        --stdin \
        --stdin-filename postgres.sql \
        --cache-dir /cache \
        --tag postgres

echo "[$(date)] Pruning old snapshots..."

docker run --rm \
  --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  -v restic-cache:/cache \
  restic/restic forget --prune \
    --cache-dir /cache \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 3

echo "[$(date)] Done."

# Telegram notification — VPS has direct access to Telegram API
curl -s --max-time 10 \
  "https://api.telegram.org/bot${TG_BOT}/sendMessage" \
  -d chat_id="${TG_CHAT}" \
  -d text="✅ Postgres backup done — $(date '+%Y-%m-%d %H:%M')" \
  > /dev/null || echo "[$(date)] Telegram notification failed, skipping"
```

```bash
chmod 700 /opt/restic/backup.sh
```

### Initialize repository (once)

```bash
docker run --rm \
  --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  restic/restic init
```

### Cron

```
0 3 * * * /opt/restic/backup.sh >> /var/log/restic-backup.log 2>&1
```

---

## Home Server — Vaultwarden Directory Backup

### File structure

```
/opt/restic/
├── .env
└── backup.sh
```

### `.env`

```bash
AWS_ACCESS_KEY_ID=YOUR_KEY
AWS_SECRET_ACCESS_KEY=YOUR_SECRET
RESTIC_REPOSITORY=s3:s3.your-provider.com/YOUR_BUCKET/home-vaultwarden
RESTIC_PASSWORD=YOUR_DIFFERENT_STRONG_PASSWORD
TG_BOT=YOUR_BOT_TOKEN
TG_CHAT=YOUR_CHAT_ID
# Optional: SOCKS5 proxy if Telegram API is blocked in your region
SOCKS5_PROXY=your-proxy-host:port
```

```bash
chmod 600 /opt/restic/.env
```

### `backup.sh`

```bash
#!/bin/bash
set -euo pipefail

source /opt/restic/.env

echo "[$(date)] Starting backup..."

# Mount the data directory read-only into the restic container
docker run --rm \
  --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  -v /path/to/vaultwarden:/data:ro \
  -v restic-cache:/cache \
  restic/restic backup /data \
    --cache-dir /cache \
    --tag vaultwarden \
    --exclude /data/icon_cache

echo "[$(date)] Pruning old snapshots..."

docker run --rm \
  --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  -v restic-cache:/cache \
  restic/restic forget --prune \
    --cache-dir /cache \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 3

echo "[$(date)] Done."

# Telegram notification
# If Telegram API is blocked, route through a SOCKS5 proxy:
# curl -s --max-time 10 --socks5-hostname "${SOCKS5_PROXY}" \
curl -s --max-time 10 \
  "https://api.telegram.org/bot${TG_BOT}/sendMessage" \
  -d chat_id="${TG_CHAT}" \
  -d text="✅ Vaultwarden backup done — $(date '+%Y-%m-%d %H:%M')" \
  > /dev/null || echo "[$(date)] Telegram notification failed, skipping"
```

```bash
chmod 700 /opt/restic/backup.sh
```

### Initialize repository (once)

```bash
docker run --rm \
  --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  restic/restic init
```

### Cron (offset by 30 minutes from the VPS job)

```
30 3 * * * /opt/restic/backup.sh >> /var/log/restic-backup.log 2>&1
```

---

## Retention Policy

| Period | Snapshots kept |
|---|---|
| Daily | 7 |
| Weekly | 4 |
| Monthly | 3 |

---

## Snapshots and Restore

### How snapshots work

Every `restic backup` run creates a snapshot — a point in time. Snapshots are independent but physically share chunks through deduplication.

```
Snapshot 1 (Mar 28) ──┐
Snapshot 2 (Mar 29) ──┼──► shared chunks in S3/data/
Snapshot 3 (Mar 30) ──┘      + new chunks for changes
```

Deleting one snapshot never affects others — `prune` only removes chunks that no snapshot references anymore.

### List snapshots

```bash
docker run --rm --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  restic/restic snapshots
```

Output:
```
ID        Time                 Host        Tags         Paths   Size
----------------------------------------------------------------------
211a4a6a  2026-03-28 14:14:45  homeserver  vaultwarden  /data   3.468 MiB
f85a0f8c  2026-03-29 03:00:28  homeserver  vaultwarden  /data   512 KiB
```

- **ID** — unique snapshot identifier, first 8 chars are enough
- **Size** — only the new data added in this snapshot, not total volume
- `latest` — alias for the most recent snapshot

### Browse snapshot contents without restoring

```bash
docker run --rm --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  restic/restic ls latest

# specific snapshot
docker run --rm --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  restic/restic ls 211a4a6a
```

### Restore — Vaultwarden

**Step 1 — stop the container first**

```bash
cd /path/to/vaultwarden
docker compose down
```

> Never restore while the container is running — you'll get write conflicts.

**Step 2 — restore from snapshot**

```bash
docker run --rm --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  -v /path/to/vaultwarden:/restore \
  restic/restic restore latest \
    --target /restore
```

For a specific snapshot instead of latest:

```bash
restic/restic restore 211a4a6a --target /restore
```

**Step 3 — start the container**

```bash
docker compose up -d
```

### Restore — PostgreSQL

The backup is a SQL dump sent via stdin, so restore goes through `psql`.

**Option A — via temp file**

```bash
docker run --rm --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  restic/restic dump latest /postgres.sql > /tmp/restore.sql

docker compose stop app   # stop the app, not the db
docker exec -i postgres-db psql -U postgres postgres < /tmp/restore.sql
docker compose start app
```

**Option B — pipe without temp file**

```bash
docker run --rm --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  restic/restic dump latest /postgres.sql \
  | docker exec -i postgres-db psql -U postgres postgres
```

### Partial restore — single file or directory

```bash
docker run --rm --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  -v /tmp/restore:/restore \
  restic/restic restore latest \
    --target /restore \
    --include /data/db-backups/
```

### Verify repository integrity

```bash
# fast check — metadata only
docker run --rm --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  restic/restic check

# full check — reads all data from S3 (slow)
docker run --rm --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  restic/restic check --read-data
```

Run `restic check` monthly to make sure backups are intact.

---

## How Restic Works Under the Hood

### Repository structure in S3

```
YOUR_BUCKET/vps-postgres/
├── config          ← repo parameters, version, ID
├── keys/           ← encrypted master keys
├── snapshots/      ← snapshot metadata (JSON files)
├── index/          ← index of all chunks
├── data/           ← actual data, split into chunks
└── locks/          ← concurrency locks
```

### Encryption

On `restic init`, a random master key is generated and encrypted with your password, then stored in `keys/`. On every run, restic decrypts the master key using `RESTIC_PASSWORD` and uses it to encrypt data. Everything in S3 is encrypted — without the password, the data is unreadable noise.

**If you lose the repository password — your backups are gone.** Store it in a password manager and keep a second copy somewhere safe.

### Deduplication via Content-Defined Chunking

Before uploading, data is split into variable-size chunks (CDC — Content-Defined Chunking). Chunk boundaries are determined by content, not file position.

Each chunk is hashed (SHA-256). If that hash already exists in the repository — the chunk is not uploaded again.

```
Snapshot 1: [chunk A] [chunk B] [chunk C]
Snapshot 2: [chunk A] [chunk B] [chunk D]  ← only chunk D is new
                                              A and B are not uploaded again
```

This is why the first backup is large and subsequent ones are small — only changed chunks are uploaded.

### Snapshot internals

A snapshot is just a **JSON metadata file** in `snapshots/`:

```json
{
  "time": "2026-03-28T03:00:00Z",
  "hostname": "my-server",
  "tags": ["postgres"],
  "paths": ["/postgres.sql"],
  "tree": "abc123..."   ← reference to the file tree in data/
}
```

The snapshot contains no data — only references to chunks in `data/`. That's why `forget` is fast (removes metadata) while `prune` is slow (scans all snapshots and deletes unreferenced chunks from S3).

### Data flow — PostgreSQL (stdin)

```
pg_dump → stdout
    │
   pipe (on host)
    │
    ▼
restic backup --stdin
    │
    ├── splits stream into chunks
    ├── hashes each chunk (SHA-256)
    ├── checks index — does this chunk already exist in S3?
    ├── encrypts new chunks with master key
    └── uploads new chunks to S3/data/

    └── writes snapshot to S3/snapshots/
```

No temp file is created — data flows as a stream directly into restic.

### Data flow — directory backup

```
/data/ (mounted :ro into container)
    │
restic backup /data
    │
    ├── walks the file tree
    ├── splits each file into chunks
    ├── compares against previous snapshot (via cache)
    ├── skips unchanged chunks
    └── uploads only new chunks to S3
```

The `:ro` flag means restic cannot modify the source data — a good safety practice.

### Cache

The `restic-cache` Docker volume stores a local copy of the repository index. Without it, restic would download the full index from S3 on every run to know which chunks already exist. With the cache, it checks locally and only goes to S3 for new data.

### `forget` vs `prune`

```
restic forget   → removes snapshot metadata (fast)
                  chunks in S3 are NOT deleted yet

restic prune    → scans all remaining snapshots,
                  finds orphaned chunks,
                  deletes them from S3 (slow)
```

Using `forget --prune` does both in one pass.

---

## S3 — Object Storage Concepts

### How object storage works

S3 is not a filesystem. There are no real folders — just a flat namespace of objects, each with a unique **key**:

```
vps-postgres/snapshots/abc123   ← one object with this name
vps-postgres/data/def456        ← another object
```

What looks like folders are just slashes in the key name. S3 can filter by prefix and display it folder-like, but physically everything is flat.

Each object consists of: the data itself, a unique key, and metadata (size, date, content-type).

**Key constraint** — you can't partially update an object. One byte changed means re-uploading the whole object. This is exactly why restic uses chunks — each chunk is a separate S3 object, so only changed chunks need to be re-uploaded.

### Bucket

A bucket is an isolated container for objects with its own access settings, storage region, and policies. Think of it less like a folder and more like a separate hard drive.

```
YOUR_BUCKET/
├── vps-postgres/      ← just a key prefix
└── home-vaultwarden/  ← just a key prefix
```

With two separate buckets you'd get true isolation — separate credentials per bucket. With one bucket and two prefixes, the same S3 key has access to everything.

### ACL and IAM Policies

ACL (Access Control List) controls who can do what with a bucket or object. The Zero Trust principle applies here — minimum permissions per key.

In practice, IAM policies are more flexible. For example, a write-only key scoped to one prefix:

```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject"],
  "Resource": "arn:aws:s3:::bucket/vps-postgres/*"
}
```

This is **append-only** access — ideal for backups. If a server is compromised and the key leaks, the attacker can only write new data, not delete existing backups or read other repositories.

### Versioning

When versioning is enabled, overwriting an object keeps the old version with a separate ID:

```
db.sqlite v1 (Mar 28) ← retained
db.sqlite v2 (Mar 29) ← retained
db.sqlite v3 (Mar 30) ← current
```

This protects against accidental deletion — you can roll back at the S3 level without restic. For our use case it's not critical since restic already handles snapshot history. It also doubles storage usage, which matters on limited plans.

---

## Disaster Recovery

### RPO and RTO

**RPO (Recovery Point Objective)** — maximum acceptable data loss measured in time. Backup runs at 3am, server dies at 10am → RPO = 7 hours, all changes in that window are lost. The more critical the data, the lower the RPO should be, and the more frequent the backups.

**RTO (Recovery Time Objective)** — maximum acceptable time to restore service after an incident.

Example: server dies at 10:00, restored and back online at 12:00 → RTO = 2 hours, RPO = 7 hours.

For a homelab running non-transactional services (password manager, personal apps), both numbers are acceptable. For production databases with continuous writes, you'd need WAL archiving to get RPO down to seconds.

### How to reduce RTO

Not through automated restore — that's risky. Automatic failover can cause **split brain**: the system thinks the primary node is dead and promotes a replica, but the primary is just temporarily unreachable. Now two nodes are writing to the same data simultaneously.

What actually helps:

**Runbook** — a document with clear recovery steps. In a stressful incident you don't want to think — you want to follow a checklist. This `.md` file is your runbook.

**Pre-configured environment** — Docker, restic, and `.env` already in place. No setup needed during an incident.

**Regular drills** — quarterly, actually restore from backup into `/tmp/restore` and verify the data is intact and the service comes up. A backup you've never tested is not a backup — it's hope.

### 3-2-1 Rule

The standard backup best practice:
- **3** copies of data
- **2** different storage types/locations
- **1** copy offsite

Current setup: data on server + backup in S3 = **2-1-1**. Good enough for a homelab. For full 3-2-1, add a second backup destination (Backblaze B2, another VPS, or a local NAS).

---

## Backup Monitoring

### Current setup

Telegram notifications are configured on both servers. After each successful backup, a message is sent. If the script crashes — `set -euo pipefail` exits before reaching the curl call, so no notification arrives, which itself signals something went wrong.

```
backup succeeds → telegram "✅ Vaultwarden backup done — 2026-04-02 03:30"
backup fails    → script exits early → no notification → you notice the silence
```

If Telegram API is blocked in your region (e.g. Russia), route the curl through a SOCKS5 proxy:

```bash
curl -s --max-time 10 --socks5-hostname your-proxy:port \
  "https://api.telegram.org/bot${TG_BOT}/sendMessage" \
  ...
```

`--max-time 10` ensures the script doesn't hang if the proxy is unreachable.

### Manual check

```bash
# check that snapshots exist and are recent
docker run --rm --dns 1.1.1.1 --env-file /opt/restic/.env restic/restic snapshots

# check last run log
tail -50 /var/log/restic-backup.log
```

### Next step — healthchecks.io

Telegram notifies on success but doesn't actively alert on failure. For complete coverage, add a heartbeat via [healthchecks.io](https://healthchecks.io) — if no ping arrives within N hours, an alert fires:

```bash
curl -fsS https://hc-ping.com/YOUR-UUID > /dev/null
```

Free tier available, self-hosted option exists.

---

## Common Mistakes

**Empty snapshot** — `pg_dump` fails (wrong user, DB unreachable), restic receives an empty stream and creates a 0-byte snapshot. Fix: `set -euo pipefail` with `pipefail` makes the pipe fail if pg_dump exits with an error, so restic never runs.

**`forget` without `prune`** — snapshot disappears from the list but S3 space is not freed. Chunks remain as orphans. Always use `forget --prune` together.

**Restoring while the container is running** — write conflicts between the running app and restic overwriting files. Always `docker compose down` before restoring.

**Losing the repository password** — data stays in S3 but becomes unreadable encrypted noise. Store the password in a password manager and keep a second copy somewhere else.

**Deleting snapshots directly in S3** — don't. Chunks become orphaned, the index breaks. Always manage snapshots through restic commands only.

---

## Security Notes

**`.env` with `chmod 600`** — reasonable practice for a homelab. Only root can read it. No need to overcomplicate with a secrets manager unless the threat model demands it.

**Single S3 key for multiple servers** — if using one set of credentials, a compromised server leaks the S3 key. The attacker cannot read encrypted repository data without the repo password, but they can delete or overwrite objects in the bucket. Acceptable risk for a homelab; for production, use per-server IAM keys scoped to their prefix with write-only permissions.

**Repo password ≠ S3 key** — two separate secrets. The S3 key grants access to the storage. The repo password decrypts the data. Even if the S3 key leaks, data is unreadable without the repo password.

---

## Quick Reference

```bash
# List snapshots
docker run --rm --dns 1.1.1.1 --env-file /opt/restic/.env restic/restic snapshots

# Check repository integrity
docker run --rm --dns 1.1.1.1 --env-file /opt/restic/.env restic/restic check

# Restore latest snapshot
docker run --rm --dns 1.1.1.1 \
  --env-file /opt/restic/.env \
  -v /tmp/restore:/restore \
  restic/restic restore latest --target /restore

# Run backup manually
/opt/restic/backup.sh

# View recent log
tail -50 /var/log/restic-backup.log
```

---

## Notes

- `--dns 1.1.1.1` — required when the host uses a VPN/mesh network whose DNS doesn't resolve external domains inside Docker containers (common with Tailscale, WireGuard-based overlays, etc.)
- `set -euo pipefail` — script exits on any error; prevents empty snapshots from being created when upstream commands fail
- `restic-cache` volume — caches the repository index locally; significantly speeds up repeated runs by avoiding full index download from S3
- Each repository has its own encryption password — credential isolation at the data level even when sharing S3 credentials
- Logs: `/var/log/restic-backup.log`
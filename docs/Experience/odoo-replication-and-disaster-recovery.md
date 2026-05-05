---
title: Odoo Replication And Disaster Recovery
deprecated: false
hidden: false
metadata:
  robots: index
---
A reusable, end-to-end guide for setting up a **read-only physical streaming
PostgreSQL replica + filestore rsync mirror** of a production Odoo instance
when prod is a Docker container reachable only via SSH.

The guide is generalized: every deployment-specific value (hostnames, ports,
container IDs, DB names) is parameterised at the top. Walk through it
top-to-bottom on a fresh Ubuntu VM. Verification commands are inline at
each step — don't proceed past a failed verify.

***

## 0 — When to use this exact pattern

Use this runbook when **all** of the following are true:

- Production runs Odoo + PostgreSQL **in a Docker container** on a remote host.
- The remote host is reachable from the replica VM **only via SSH** (no direct
  Postgres TCP, no VPN). Replication will tunnel through SSH.
- You have or can obtain **passwordless sudo** on the prod host, and a
  **PostgreSQL superuser** that can read prod's settings + create roles.
- You want **byte-for-byte physical streaming replication** (same major
  version, read-only standby) — not logical replication, not periodic dumps.
- You want the **filestore mirrored**, not just the DB. Without the
  filestore, attachments resolve to `None` on the replica.

If prod and replica major versions differ, see `## Mismatched majors` below.
If you don't need attachments, skip Section B.

***

## 1 — Variables (fill these in once)

Read these out of prod first; **don't assume them**. Many gotchas in this
session came from wrong assumptions about versions and roles.

```bash
# === REPLICA SIDE ===
REPLICA_HOSTNAME=$(hostname -s)            # e.g. prd-vnic
REPLICA_TUNNEL_PORT=40000                  # local port for the tunnel; usually
                                           # match prod's exposed port

# === PROD SIDE ===
PROD_SSH_USER=ubuntu
PROD_SSH_HOST=host_name_or_ip
PROD_SSH_PORT=22
PROD_PG_LOOPBACK_PORT=PG_PORT                # port the prod container exposes on
                                           # the prod host's localhost

# === DATABASE ===
PG_MAJOR=$PG_VERSION                                # MUST match prod's major. Verify.
ODOO_DB=$DATABASE_NAME
PROD_PG_SUPERUSER=$PG_USERNAME                     # often "odoo", sometimes "postgres".
                                           # Verify: rolsuper=t, rolreplication=t.

# === REPLICATION ===
REPL_USER=replicator                       # role we'll create on prod
SLOT_NAME=replica_${REPLICA_HOSTNAME//-/_} # one slot per replica; underscores only

# === FILESTORE ===
# The host path on prod where the docker volume's filestore lives.
# Discover it once with: docker inspect <odoo-container> | jq '.[0].Mounts'
PROD_FILESTORE_HOST_PATH=/var/lib/docker/volumes/<project>_<volume>/_data/filestore
LOCAL_FILESTORE_BASE=/var/lib/odoo/filestore
```

Save the filled-in values somewhere; they show up in many commands. The rest
of this guide assumes they're available as shell vars in your session.

***

## 2 — Pre-flight on prod (read-only checks)

Before touching anything, confirm the prod side is set up the way you think
it is. Run these with whatever access you have to prod (`ssh` + `sudo`):

```bash
# Versions: prod's PostgreSQL major must equal your planned PG_MAJOR.
# Connect via prod's existing path (e.g. docker exec, or once the tunnel is
# up — Section A). For now, ssh in:
ssh ${PROD_SSH_USER}@${PROD_SSH_HOST} \
  'docker ps --format "{{.Names}}\t{{.Image}}" | grep -i odoo'

# Find the Odoo container's data volume host path:
ssh ${PROD_SSH_USER}@${PROD_SSH_HOST} \
  "sudo docker inspect <odoo-container> --format '{{json .Mounts}}' | jq"

# Filestore size (so you know how big the seed will be):
ssh ${PROD_SSH_USER}@${PROD_SSH_HOST} \
  "sudo du -sh ${PROD_FILESTORE_HOST_PATH}/${ODOO_DB}"

# Can ubuntu@prod sudo without password? (Required for filestore rsync.)
ssh ${PROD_SSH_USER}@${PROD_SSH_HOST} 'sudo -n true && echo SUDO_OK || echo NO_SUDO'
```

**Mismatched majors.** If prod is a major you can't run on the replica
(e.g. prod=13, you wanted 15), STOP. You must either downgrade the planned
replica, upgrade prod, or pivot to logical replication. Physical streaming
**will not** cross majors.

***

# Section A — PostgreSQL physical streaming replica via SSH tunnel

## A1 — Install PostgreSQL \<PG\_MAJOR> + autossh on the replica

PG packages older than the latest are typically not in the default Ubuntu
archive — use the PGDG repo.

```bash
sudo apt-get update
sudo apt-get install -y curl ca-certificates gnupg lsb-release autossh

# Add PGDG repo (idempotent)
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc \
  -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc
echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  | sudo tee /etc/apt/sources.list.d/pgdg.list >/dev/null
sudo apt-get update

sudo apt-get install -y postgresql-${PG_MAJOR} postgresql-client-${PG_MAJOR}
```

**Verify:**

```bash
pg_lsclusters                                   # ${PG_MAJOR}/main, online
sudo -u postgres psql -c 'SELECT version();'    # major matches PG_MAJOR
which autossh                                   # /usr/bin/autossh
```

If a different major was already installed by mistake, drop it cleanly:

```bash
sudo systemctl stop postgresql@<old>-main
sudo pg_dropcluster --stop <old> main
sudo apt-get purge -y postgresql-<old> postgresql-client-<old>
sudo rm -rf /etc/systemd/system/postgresql@<old>-main.service.d
sudo systemctl daemon-reload
```

## A2 — Persistent SSH tunnel (autossh + systemd)

The tunnel must be up **before** Postgres starts, must auto-reconnect, and
must outlive any single SSH session. It runs as the `postgres` system user
under a dedicated systemd unit.

### A2.1 SSH key for `postgres` and authorize on prod

```bash
sudo -u postgres mkdir -p ~postgres/.ssh && sudo -u postgres chmod 700 ~postgres/.ssh
sudo -u postgres ssh-keygen -t ed25519 -N '' \
  -f ~postgres/.ssh/id_ed25519 -C "pg-replica-tunnel@${REPLICA_HOSTNAME}"
sudo -u postgres cat ~postgres/.ssh/id_ed25519.pub
```

Add the printed pubkey to `${PROD_SSH_USER}@${PROD_SSH_HOST}:~/.ssh/authorized_keys`
on prod. Either use your existing access:

```bash
sudo -u postgres cat ~postgres/.ssh/id_ed25519.pub | \
  ssh ${PROD_SSH_USER}@${PROD_SSH_HOST} \
    'umask 077; mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'
```

…or paste it manually.

**Verify (also accepts the host key once):**

```bash
sudo -u postgres ssh -o StrictHostKeyChecking=accept-new -o BatchMode=yes \
  ${PROD_SSH_USER}@${PROD_SSH_HOST} 'echo ssh-ok'
```

### A2.2 Tunnel systemd unit

```bash
sudo tee /etc/systemd/system/pg-replica-tunnel.service >/dev/null <<EOF
[Unit]
Description=Persistent SSH tunnel to prod Postgres for PG${PG_MAJOR} replica
After=network-online.target
Wants=network-online.target
Before=postgresql@${PG_MAJOR}-main.service

[Service]
User=postgres
Group=postgres
Environment=AUTOSSH_GATETIME=0
ExecStart=/usr/bin/autossh -M 0 -N \\
  -o ServerAliveInterval=15 \\
  -o ServerAliveCountMax=3 \\
  -o ExitOnForwardFailure=yes \\
  -o StrictHostKeyChecking=accept-new \\
  -L 127.0.0.1:${REPLICA_TUNNEL_PORT}:localhost:${PROD_PG_LOOPBACK_PORT} \\
  -p ${PROD_SSH_PORT} ${PROD_SSH_USER}@${PROD_SSH_HOST}
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### A2.3 Bind Postgres to the tunnel

So Postgres won't start without the tunnel and gets restarted with it.

```bash
sudo install -d /etc/systemd/system/postgresql@${PG_MAJOR}-main.service.d
sudo tee /etc/systemd/system/postgresql@${PG_MAJOR}-main.service.d/tunnel.conf >/dev/null <<EOF
[Unit]
Requires=pg-replica-tunnel.service
After=pg-replica-tunnel.service
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now pg-replica-tunnel.service
```

**Verify:**

```bash
sudo systemctl status pg-replica-tunnel.service
sudo ss -tlnp | grep ${REPLICA_TUNNEL_PORT}     # autossh listening on 127.0.0.1
sudo -u postgres psql "host=127.0.0.1 port=${REPLICA_TUNNEL_PORT} \
  user=${PROD_PG_SUPERUSER} dbname=postgres" -c 'SELECT version();'
# (will prompt for password; you'll fix that below)
```

## A3 — Configure the primary

Connect to prod via the tunnel as the prod superuser. Easiest is to put the
prod password into a separate `.pgpass.prod` so nothing leaks into shell
history:

```bash
sudo -u postgres bash -c 'umask 077; cat > /var/lib/postgresql/.pgpass.prod' <<EOF
127.0.0.1:${REPLICA_TUNNEL_PORT}:*:${PROD_PG_SUPERUSER}:<PASTE_PROD_PASSWORD_HERE>
EOF
```

### A3.1 Inspect prod's replication-relevant settings

```bash
sudo -u postgres -E env PGPASSFILE=/var/lib/postgresql/.pgpass.prod \
  psql -h 127.0.0.1 -p ${REPLICA_TUNNEL_PORT} -U ${PROD_PG_SUPERUSER} -d postgres -At <<'SQL'
SELECT name||' = '||setting FROM pg_settings
 WHERE name IN ('wal_level','max_wal_senders','max_replication_slots',
                'listen_addresses','archive_mode','hot_standby',
                'data_checksums','lc_collate','lc_ctype','server_encoding',
                'max_connections','max_worker_processes',
                'max_locks_per_transaction','max_prepared_transactions');
SQL
```

Required for replication on prod:

| Setting                 | Required               |
| ----------------------- | ---------------------- |
| `wal_level`             | `replica` or `logical` |
| `max_wal_senders`       | ≥ existing senders + 1 |
| `max_replication_slots` | ≥ existing slots + 1   |

If any are missing or too low, edit prod's `postgresql.conf` (in the
container image or via a docker volume), then **restart prod** (a
`wal_level` change requires a restart, not a reload).

### A3.2 Create the replication role + slot

This avoids the variable-expansion gotcha that bit us once: pass the password
through psql's `:'var'` substitution rather than embedding `$PASS` in a
single-quoted heredoc.

```bash
REPL_PASS=$(openssl rand -base64 32 | tr -d '=+/\n' | cut -c1-32)
echo "Generated replicator password (length=${#REPL_PASS})"

# Persist for replica's use; both wildcard and replication-specific entries.
sudo -u postgres bash -c "umask 077; {
  echo '127.0.0.1:${REPLICA_TUNNEL_PORT}:replication:${REPL_USER}:${REPL_PASS}'
  echo '127.0.0.1:${REPLICA_TUNNEL_PORT}:*:${REPL_USER}:${REPL_PASS}'
} >> ~/.pgpass; chmod 600 ~/.pgpass"

# Create role + slot on prod via the tunnel
sudo -u postgres -E env REPL_PASS="$REPL_PASS" PGPASSFILE=/var/lib/postgresql/.pgpass.prod \
  psql -h 127.0.0.1 -p ${REPLICA_TUNNEL_PORT} -U ${PROD_PG_SUPERUSER} -d postgres \
       -v ON_ERROR_STOP=1 -v repl_pass="$REPL_PASS" -v slot="${SLOT_NAME}" \
       -v repl_user="${REPL_USER}" <<'SQL'
DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = :'repl_user') THEN
    EXECUTE format('CREATE ROLE %I WITH REPLICATION LOGIN PASSWORD %L',
                   :'repl_user', :'repl_pass');
  ELSE
    EXECUTE format('ALTER ROLE %I WITH REPLICATION LOGIN PASSWORD %L',
                   :'repl_user', :'repl_pass');
  END IF;
END $$;

SELECT pg_create_physical_replication_slot(:'slot')
 WHERE NOT EXISTS (SELECT 1 FROM pg_replication_slots WHERE slot_name = :'slot');
SQL
```

**Verify replication-protocol auth + slot creation:**

```bash
sudo -u postgres -E env PGPASSWORD="$REPL_PASS" \
  psql "host=127.0.0.1 port=${REPLICA_TUNNEL_PORT} user=${REPL_USER} dbname=postgres replication=database" \
  -c "IDENTIFY_SYSTEM;"
# Should print a system identifier row.
```

If `IDENTIFY_SYSTEM` succeeds, prod's `pg_hba.conf` already permits the
container's view of your tunnel source IP (commonly the docker bridge
gateway, e.g. `172.18.0.1`) for `replication`. **No pg\_hba edit needed.**
If it fails with "no pg\_hba.conf entry for replication connection", check
which IP the container sees you as:

```sql
-- run on prod via the tunnel
SELECT inet_client_addr();
```

…and add a matching `host replication ${REPL_USER} <that-cidr> scram-sha-256`
line to prod's `pg_hba.conf`, then `SELECT pg_reload_conf();`.

## A4 — Take the base backup

```bash
sudo systemctl stop postgresql@${PG_MAJOR}-main
sudo -u postgres rm -rf /var/lib/postgresql/${PG_MAJOR}/main
sudo -u postgres mkdir -p /var/lib/postgresql/${PG_MAJOR}/main
sudo chmod 0700 /var/lib/postgresql/${PG_MAJOR}/main

# Run under systemd-run so it survives the SSH session (long backups).
sudo systemd-run --unit=pg-basebackup-seed --collect \
  --uid=postgres --gid=postgres \
  --working-directory=/var/lib/postgresql \
  --setenv=PGPASSWORD="$REPL_PASS" \
  -- /usr/lib/postgresql/${PG_MAJOR}/bin/pg_basebackup \
       -h 127.0.0.1 -p ${REPLICA_TUNNEL_PORT} -U ${REPL_USER} \
       -D /var/lib/postgresql/${PG_MAJOR}/main \
       -Fp -Xs -P -R -v \
       -S ${SLOT_NAME}
```

**Watch progress:**

```bash
sudo systemctl status pg-basebackup-seed.service
sudo journalctl -u pg-basebackup-seed.service -f
sudo du -sh /var/lib/postgresql/${PG_MAJOR}/main
```

When `systemctl is-active pg-basebackup-seed.service` returns `inactive`
with `Result=success`, the seed is done.

`-Fp` plain copy, `-Xs` streams WAL during the copy (no archive needed),
`-R` writes `standby.signal` + `primary_conninfo` automatically, `-S`
binds the backup to the slot we created. **No&#x20;**`-C` — slot already exists.

## A5 — Standby tuning + start

### A5.1 Clean up `primary_conninfo`

`pg_basebackup -R` embeds the password inline. Replace with `passfile`:

```bash
sudo -u postgres tee /var/lib/postgresql/${PG_MAJOR}/main/postgresql.auto.conf >/dev/null <<EOF
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
primary_conninfo = 'user=${REPL_USER} passfile=''/var/lib/postgresql/.pgpass'' host=127.0.0.1 port=${REPLICA_TUNNEL_PORT} application_name=${REPLICA_HOSTNAME}_replica'
primary_slot_name = '${SLOT_NAME}'
EOF
```

### A5.2 Match GUCs that must be ≥ master

These standby GUCs **must equal or exceed** prod, or hot standby refuses to
start:

- `max_connections`
- `max_worker_processes`
- `max_locks_per_transaction`
- `max_prepared_transactions`
- `max_wal_senders`

Drop them in via a `conf.d` file so we don't touch the package's main
`postgresql.conf`:

```bash
# Read prod values (already shown in A3.1) and mirror them here.
sudo tee /etc/postgresql/${PG_MAJOR}/main/conf.d/replica.conf >/dev/null <<'EOF'
# Required ≥ master for hot standby
max_connections = 200
max_worker_processes = 16
# Reasonable defaults for a read-only standby
hot_standby = on
hot_standby_feedback = on
EOF
```

### A5.3 Locale match

If prod uses `en_US.utf8` but your replica's OS only has `C.UTF-8`,
generate the missing locale once:

```bash
sudo locale-gen en_US.UTF-8
locale -a | grep en_US
```

### A5.4 Start the cluster

```bash
sudo systemctl start postgresql@${PG_MAJOR}-main
sudo systemctl status postgresql@${PG_MAJOR}-main --no-pager
sudo tail -200 /var/log/postgresql/postgresql-${PG_MAJOR}-main.log
```

Look for:

- `entering standby mode`
- `consistent recovery state reached`
- `started streaming WAL from primary at <LSN>`

If you instead see `FATAL: hot standby is not possible because
max_connections = X is a lower setting than on the master server`, raise
the conf.d value to match.

## A6 — Verification

```bash
# Standby side, via TCP (peer auth on socket may not match prod role names)
sudo -u postgres -E env PGPASSFILE=/var/lib/postgresql/.pgpass.prod \
  psql -h 127.0.0.1 -p 5432 -U ${PROD_PG_SUPERUSER} -d postgres -c "
SELECT pg_is_in_recovery() AS in_recovery,
       pg_last_wal_receive_lsn() AS recv,
       pg_last_wal_replay_lsn()  AS replay,
       now() - pg_last_xact_replay_timestamp() AS replay_lag;"

# Read-only enforcement
sudo -u postgres -E env PGPASSFILE=/var/lib/postgresql/.pgpass.prod \
  psql -h 127.0.0.1 -p 5432 -U ${PROD_PG_SUPERUSER} -d postgres \
  -c "CREATE TABLE __replica_write_test();"   # MUST fail "read-only transaction"

# Primary side, through the tunnel
sudo -u postgres -E env PGPASSFILE=/var/lib/postgresql/.pgpass.prod \
  psql -h 127.0.0.1 -p ${REPLICA_TUNNEL_PORT} -U ${PROD_PG_SUPERUSER} -d postgres -c "
SELECT application_name, client_addr, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_bytes_behind
  FROM pg_stat_replication
 WHERE application_name = '${REPLICA_HOSTNAME}_replica';"
```

Expected: `in_recovery=t`, `replay_bytes_behind` near 0, `state=streaming`,
the write-test fails with `cannot execute CREATE TABLE in a read-only
transaction`.

### Tunnel resilience smoke test

```bash
sudo systemctl restart pg-replica-tunnel.service
# wait ~10s — WAL receiver auto-reconnects, status returns to 'streaming'
```

***

# Section B — Filestore rsync replication

Odoo's DB only stores attachment metadata + content hashes. The actual blobs
live in the filestore directory. This section mirrors the filestore from
prod with rsync over SSH on a 5-minute cron.

## B1 — Local odoo user + paths

uid 999 is what the Odoo container uses. gid 999 is taken by
`systemd-journal` on Ubuntu by default — that's fine, we'll remap via rsync.

```bash
sudo groupadd --system odoo
sudo useradd --system --uid 999 --gid odoo \
  --home-dir /var/lib/odoo --shell /usr/sbin/nologin \
  --comment "Odoo filestore mirror" odoo
sudo install -d -o odoo -g odoo -m 0750 /var/lib/odoo /var/lib/odoo/filestore
sudo install -d -o odoo -g odoo -m 0700 /var/lib/odoo/.ssh
```

## B2 — SSH key for odoo, authorize on prod

Use a _separate_ key from the postgres tunnel one — different blast radius,
easier to rotate independently.

```bash
sudo -u odoo ssh-keygen -t ed25519 -N '' \
  -f /var/lib/odoo/.ssh/id_ed25519 \
  -C "odoo-filestore-rsync@${REPLICA_HOSTNAME}"
sudo -u odoo cat /var/lib/odoo/.ssh/id_ed25519.pub

# Push it (uses your existing prod access)
sudo -u odoo cat /var/lib/odoo/.ssh/id_ed25519.pub | \
  ssh ${PROD_SSH_USER}@${PROD_SSH_HOST} \
    'umask 077; mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'
```

**Verify:**

```bash
sudo -u odoo ssh -o StrictHostKeyChecking=accept-new -o BatchMode=yes \
  ${PROD_SSH_USER}@${PROD_SSH_HOST} \
  'echo ssh-ok; sudo -n true && echo sudo-ok || echo NO_SUDO'
```

`sudo-ok` is essential — the rsync needs `--rsync-path="sudo rsync"` to read
the root-owned docker volume.

## B3 — Initial seed (under systemd-run, survives session close)

For a 100+ GB initial seed, **do not** run inside your shell — closing the
terminal will kill it. Use `systemd-run` so it's parented to PID 1.

```bash
sudo install -d -o odoo -g odoo -m 0750 ${LOCAL_FILESTORE_BASE}/${ODOO_DB}

sudo tee /usr/local/sbin/odoo-filestore-seed.sh >/dev/null <<EOF
#!/usr/bin/env bash
set -uo pipefail
export HOME=/var/lib/odoo
exec /usr/bin/rsync -aHS --numeric-ids --partial --info=stats1 \\
  --rsync-path="sudo rsync" \\
  --usermap='*:odoo' --groupmap='*:odoo' \\
  -e "ssh -o BatchMode=yes -o ServerAliveInterval=15 -o ServerAliveCountMax=3" \\
  "${PROD_SSH_USER}@${PROD_SSH_HOST}:${PROD_FILESTORE_HOST_PATH}/${ODOO_DB}/" \\
  "${LOCAL_FILESTORE_BASE}/${ODOO_DB}/"
EOF
sudo chmod 0755 /usr/local/sbin/odoo-filestore-seed.sh

sudo systemd-run --unit=odoo-filestore-seed --collect \
  --uid=odoo --gid=odoo \
  --working-directory=/var/lib/odoo \
  /usr/local/sbin/odoo-filestore-seed.sh
```

**Watch progress:**

```bash
sudo systemctl status odoo-filestore-seed.service
sudo journalctl -u odoo-filestore-seed.service -f
sudo du -sh ${LOCAL_FILESTORE_BASE}/${ODOO_DB}
```

When `systemctl is-active odoo-filestore-seed.service` is `inactive` and
the journal's last line shows `Deactivated successfully`, the seed is done.
`--collect` removes the unit on exit so it doesn't linger.

## B4 — Cron-driven recurring sync

Wrapper script with single-flight lock, low CPU/IO priority:

```bash
sudo tee /usr/local/sbin/odoo-filestore-rsync.sh >/dev/null <<EOF
#!/usr/bin/env bash
set -euo pipefail

DB="${ODOO_DB}"
PROD_HOST="${PROD_SSH_USER}@${PROD_SSH_HOST}"
SRC="${PROD_FILESTORE_HOST_PATH}/\${DB}/"
DST="${LOCAL_FILESTORE_BASE}/\${DB}/"
LOG="/var/log/odoo-filestore-rsync.log"
LOCK="/var/lib/odoo/.rsync.lock"   # NOT /run/* — odoo can't write there

exec >>"\$LOG" 2>&1
exec 9>"\$LOCK"
flock -n 9 || { echo "\$(date -Is) skipped: previous rsync still running"; exit 0; }

echo "\$(date -Is) start"
ionice -c2 -n7 nice -n10 \\
rsync -aHS --numeric-ids \\
  --rsync-path="sudo rsync" \\
  --usermap='*:odoo' --groupmap='*:odoo' \\
  --partial \\
  -e "ssh -o BatchMode=yes -o ServerAliveInterval=15 -o ServerAliveCountMax=3" \\
  "\${PROD_HOST}:\${SRC}" "\${DST}"
rc=\$?
echo "\$(date -Is) end rc=\$rc"
exit \$rc
EOF
sudo chmod 0755 /usr/local/sbin/odoo-filestore-rsync.sh
sudo chown root:root /usr/local/sbin/odoo-filestore-rsync.sh

sudo install -m 0640 -o odoo -g odoo /dev/null /var/log/odoo-filestore-rsync.log

# Cron entry
sudo tee /etc/cron.d/odoo-filestore-rsync >/dev/null <<'EOF'
# Replicate Odoo filestore from prod every 5 minutes.
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
*/5 * * * * odoo /usr/local/sbin/odoo-filestore-rsync.sh
EOF
sudo chmod 0644 /etc/cron.d/odoo-filestore-rsync
```

**Important — only install the cron entry AFTER the initial seed has
finished.** Otherwise the cron will fire every 5 min and try to sync against
a destination that's still being seeded, making both runs fight. The flock
in the wrapper protects future cron-vs-cron runs but the seed in B3 doesn't
hold it.

**Verify:**

```bash
# Manual run as odoo (the cron will use the same path)
sudo -u odoo /usr/local/sbin/odoo-filestore-rsync.sh
sudo tail -5 /var/log/odoo-filestore-rsync.log    # last line should be `end rc=0`

# Counts/sizes match prod (small drift is normal — files added during the run)
sudo find ${LOCAL_FILESTORE_BASE}/${ODOO_DB} -type f | wc -l
sudo -u odoo ssh ${PROD_SSH_USER}@${PROD_SSH_HOST} \
  "sudo find ${PROD_FILESTORE_HOST_PATH}/${ODOO_DB} -type f | wc -l"
```

***

# Section C — Operations

## Daily health check (one liner)

```bash
systemctl is-active pg-replica-tunnel.service postgresql@${PG_MAJOR}-main.service && \
sudo ss -tln | grep -q :${REPLICA_TUNNEL_PORT} && \
[ -n "$(find /var/log/odoo-filestore-rsync.log -mmin -15)" ] && \
echo "all green" || echo "investigate"
```

## Reading logs

```bash
# Tunnel
journalctl -u pg-replica-tunnel.service -f

# Postgres
sudo tail -f /var/log/postgresql/postgresql-${PG_MAJOR}-main.log

# Filestore rsync
sudo tail -f /var/log/odoo-filestore-rsync.log
```

## Replication-state queries

```bash
# Standby — through TCP, password from .pgpass.prod
sudo -u postgres -E env PGPASSFILE=/var/lib/postgresql/.pgpass.prod \
  psql -h 127.0.0.1 -p 5432 -U ${PROD_PG_SUPERUSER} -d postgres <<'SQL'
SELECT pg_is_in_recovery(),
       pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn(),
       now() - pg_last_xact_replay_timestamp() AS replay_lag;
SELECT status, sender_host, sender_port, slot_name,
       latest_end_lsn, last_msg_receipt_time
  FROM pg_stat_wal_receiver;
SQL

# Primary — through the tunnel
sudo -u postgres -E env PGPASSFILE=/var/lib/postgresql/.pgpass.prod \
  psql -h 127.0.0.1 -p ${REPLICA_TUNNEL_PORT} -U ${PROD_PG_SUPERUSER} -d postgres <<SQL
SELECT application_name, client_addr, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_bytes_behind
  FROM pg_stat_replication
 WHERE application_name = '${REPLICA_HOSTNAME}_replica';
SELECT slot_name, active, restart_lsn,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_bytes
  FROM pg_replication_slots WHERE slot_name = '${SLOT_NAME}';
SQL
```

## What "healthy" looks like

| Signal                                        | Healthy                  | Trouble                       |
| --------------------------------------------- | ------------------------ | ----------------------------- |
| `pg-replica-tunnel.service`                   | `active (running)`       | failed / restarting in a loop |
| listener on `:${REPLICA_TUNNEL_PORT}`         | one autossh-owned socket | absent                        |
| `pg_stat_wal_receiver.status` (standby)       | `streaming`              | empty row, `disconnected`     |
| `pg_stat_replication.state` (primary)         | `streaming`              | absent row                    |
| `replay_bytes_behind` (primary)               | small / 0                | growing without recovery      |
| `pg_replication_slots.active` (primary)       | `t`                      | `f` for >a few sec            |
| `pg_replication_slots` retained\_bytes        | small / stable           | growing — long tunnel outage  |
| `/var/log/odoo-filestore-rsync.log` last line | recent + `rc=0`          | older than 15 min, or `rc!=0` |

## Promotion (failover)

If prod is gone and you need this replica to take over writes:

```bash
sudo -u postgres pg_ctlcluster ${PG_MAJOR} main promote
```

Once promoted, the cluster diverges from prod and **cannot become a standby
again without a fresh&#x20;**`pg_basebackup`. Don't promote casually — do it only
during real failover.

## Decommission

If you tear this replica down:

```bash
# 1. Stop services here
sudo systemctl disable --now postgresql@${PG_MAJOR}-main pg-replica-tunnel.service
# 2. Drop the slot on prod (otherwise prod retains WAL forever)
sudo -u postgres -E env PGPASSFILE=/var/lib/postgresql/.pgpass.prod \
  psql -h 127.0.0.1 -p ${REPLICA_TUNNEL_PORT} -U ${PROD_PG_SUPERUSER} -d postgres \
  -c "SELECT pg_drop_replication_slot('${SLOT_NAME}');"
# 3. Remove the authorized_keys entries on prod (postgres + odoo pubkeys)
# 4. Remove the cron + wrappers
sudo rm -f /etc/cron.d/odoo-filestore-rsync \
           /usr/local/sbin/odoo-filestore-{seed,rsync}.sh \
           /etc/systemd/system/pg-replica-tunnel.service \
           /etc/systemd/system/postgresql@${PG_MAJOR}-main.service.d/tunnel.conf
sudo systemctl daemon-reload
```

***

# Section D — Gotchas we actually hit (read this!)

These are non-obvious traps from real deployments. Each one cost real time
to debug.

1. **PG version mismatch.** Always run `SELECT version();` on prod _before_
   installing on the replica. Physical streaming requires identical major;
   "we use PG15 in dev so prod must be 15 too" is not a check.

2. **Locale availability.** prod's cluster locale (e.g. `en_US.utf8`) must
   exist on the replica's OS. `sudo locale-gen en_US.UTF-8` if not.

3. **Variable expansion through&#x20;**`sudo bash -c '...'`**.** Single-quoted heredocs
   don't expand `$VAR` from the _outer_ shell. We once created a role with an
   empty password because the password var didn't reach the inner bash. Use
   `psql -v var="$VAR"` substitution instead, or pass via `sudo -E env VAR=...`.

4. `.pgpass`**&#x20;lookup with replication conns.** For `replication=database`,
   libpq's pgpass lookup uses dbname=`replication`, but some client paths
   look up `*` instead. Add **both** entries to `~postgres/.pgpass`:
   `host:port:replication:user:pass` AND `host:port:*:user:pass`.

5. `max_connections`**&#x20;(and friends) ≥ master.** PG13+ refuses to start the
   standby otherwise — error is loud, but startup just dies. Mirror prod's
   value via `/etc/postgresql/<v>/main/conf.d/replica.conf`.

6. `hot_standby_feedback`**.** Prevents standby query cancellation due to
   prod-side vacuum; but pins tuples on prod. On for reporting replicas, off
   for pure DR.

7. **Lock file paths.** Cron-driven scripts that run as a non-root user
   (e.g. `odoo`) can't write to `/run/`. Use `/var/lib/<app>/.lock`.

8. **Don't enable cron until the seed is done.** The cron's flock won't
   protect against the seed (the seed isn't holding the lock). Either install
   cron after the seed, or have the seed take the same flock.

9. **Long-running operations need a system-wide parent.** `pg_basebackup`
   (often >1h) and the filestore initial seed (often >1h) **die when your
   shell closes** if you start them from a normal session. Use
   `systemd-run --unit=<name> --collect`.

10. **Embedded password in&#x20;**`primary_conninfo`**.** `pg_basebackup -R` writes
    the password inline into `postgresql.auto.conf`. Replace with `passfile=`
    pointing at `~postgres/.pgpass`.

11. **Docker bridge IP on prod.** When the prod Postgres is in a non-host
    container, your tunneled connection arrives at the container as the
    bridge gateway (e.g. `172.18.0.1`), **not** `127.0.0.1`. `pg_hba.conf`
    on prod must permit that CIDR for `replication`. Run
    `SELECT inet_client_addr()` to confirm.

12. `sudo rsync`**&#x20;on prod.** Filestores under `/var/lib/docker/volumes/`
    are root-owned. The rsync that runs on prod needs sudo: pass
    `--rsync-path="sudo rsync"`. This requires `ubuntu@prod` to have
    passwordless sudo (most cloud-init Ubuntu hosts do).

13. **uid/gid mapping for filestore.** Prod's container runs Odoo as
    uid:gid `999:999`. Locally, gid 999 is usually `systemd-journal` on
    Ubuntu. Don't fight it — pin uid 999 for the local `odoo` user, accept
    whatever gid `groupadd` gives the `odoo` group, and use rsync's
    `--usermap='*:odoo' --groupmap='*:odoo'` to remap by name on the wire.

14. **Don't add&#x20;**`--delete`**&#x20;to filestore rsync until you've watched it for
    a few days.** Odoo prunes orphan attachments rarely; an early `--delete`
    can wipe files the standby still references via cached metadata.

15. **One slot per replica.** Don't reuse a slot name across replicas —
    each consumer needs its own. Encode the replica's hostname in the slot
    name so it's unambiguous in `pg_replication_slots`.

<br />

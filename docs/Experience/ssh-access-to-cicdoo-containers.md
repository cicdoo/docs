---
title: SSH Access to CICDoo Containers
deprecated: false
hidden: false
metadata:
  robots: index
---
How to open a shell inside any CICDoo instance container (Odoo or Postgres) running on a remote server, from your local machine.

## Container naming

Each instance creates two containers, named after the instance UUID:

| Service  | Container name       |
| -------- | -------------------- |
| Odoo     | `<instance-uuid>`    |
| Postgres | `<instance-uuid>_db` |

Example for instance `3c8c3b40-fb55-4ee3-809d-b181c45e700d`:

- Odoo: `3c8c3b40-fb55-4ee3-809d-b181c45e700d`
- DB:   `3c8c3b40-fb55-4ee3-809d-b181c45e700d_db`

You can list them on the server with `docker ps`.

## Prerequisites

- SSH access to the server hosting the instance (key file + user with `sudo`).
- Local Docker CLI is **not** required for the alias method below.

## Quick method — alias + sudo over SSH

Add to your `~/.zshrc` or `~/.bashrc`, replacing the key path, user, and host:

```bash
alias rdocker-it='ssh -t -i ~/path/to/key USERNAME@HOST_IP sudo docker'
```

Reload the shell (`source ~/.zshrc`) and use it like a local docker:

```bash
# List running containers
rdocker-it ps

# Shell into an Odoo container
rdocker-it exec -it <instance-uuid> bash

# Shell into the matching Postgres container
rdocker-it exec -it <instance-uuid>_db bash

# psql directly inside the DB container
rdocker-it exec -it <instance-uuid>_db psql -U odoo -d <database-name>

# Tail Odoo logs from outside the container (logs are bind-mounted on the host)
rdocker-it exec -it <instance-uuid> tail -f /var/log/odoo/odoo.log
```

The `-t` on `ssh` is required so that interactive flags (`-it`) get a real TTY.

If you manage multiple servers, define one alias per server:

```bash
alias rdocker-milan='ssh -t -i ~/Downloads/milan.pem ubuntu@15.160.109.63 sudo docker'
alias rdocker-fra='ssh   -t -i ~/Downloads/fra.pem    ubuntu@10.0.0.42       sudo docker'
```

## Alternative — Docker context over SSH (no alias)

If your local user can run `docker` on the remote without `sudo` (i.e. is in the remote `docker` group), you can use a real Docker context and all docker tooling — `docker exec`, `docker compose`, IDE Docker plugins — works against the remote daemon:

```bash
# One-time setup
docker context create milan --docker "host=ssh://ubuntu@15.160.109.63"

# Use it
docker context use milan
docker ps
docker exec -it <instance-uuid> bash

# Switch back to your local Docker
docker context use default
```

For SSH key selection, pin it via `~/.ssh/config`:

```text
Host 15.160.109.63
    User ubuntu
    IdentityFile ~/Downloads/milan.pem
    IdentitiesOnly yes
```

If `ubuntu` is **not** in the remote `docker` group, this method fails with `permission denied` on `/var/run/docker.sock`. Either add the user to the group on the server (`sudo usermod -aG docker ubuntu`, then re-login) or fall back to the alias method above.

## Common in-container tasks

Once you're inside an Odoo container (`rdocker-it exec -it <uuid> bash`):

```bash
# Odoo source paths
ls /opt/odoo/ce            # community
ls /opt/odoo/ee            # enterprise (if EE)
ls /opt/odoo/themes
ls /var/lib/odoo           # filestore
ls /var/log/odoo           # logs
ls /var/backups            # backups

# Restart Odoo from inside (PID 1 is the entrypoint loop)
pkill -f 'odoo-bin' && tail -f /var/log/odoo/odoo.log
```

Inside the DB container:

```bash
psql -U odoo -d <database-name>
pg_dump -U odoo -d <database-name> -Fc -f /tmp/dump.dump
```

## Troubleshooting

`channel N: open failed: connect failed: open failed`
SSH could open the connection but couldn't reach `/var/run/docker.sock` on the remote. The user isn't in the `docker` group. Either add it (`sudo usermod -aG docker <user>` and re-login) or use the `sudo`-based alias method.

`error during connect: ... EOF`
Local socket exists but the SSH tunnel died or was never established. Check the tunnel process is running, then `rm -f` the stale local socket and reconnect with `-o StreamLocalBindUnlink=yes`.

`the input device is not a TTY`
Missing `-t` on the SSH side. Use `ssh -t ...` (the alias above already includes it).

**Container name not found**
Run `rdocker-it ps --format '{% raw %}{{.Names}}{% endraw %}'` to see exact names. Stopped containers don't appear unless you add `-a`.

**Permission denied on key file**
`chmod 600 ~/Downloads/milan.pem`.

## Security notes

- The remote Docker daemon is root-equivalent. Anyone with shell access as a `docker`-group user (or `sudo docker`) can take over the host. Keep key files private (`chmod 600`) and avoid sharing aliases that hard-code them.
- Never expose the Docker daemon over plain TCP on a public interface. If you ever bridge it (e.g. via `socat`), bind to `127.0.0.1` only and reach it through an SSH tunnel.

<br />

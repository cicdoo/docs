---
title: Instances & the Console
deprecated: false
hidden: false
metadata:
  robots: index
---
An **instance** is a running Odoo environment deployed on a server, associated with a specific branch of a project. Each instance has its own database, filestore, and Docker container stack.

### Console Overview

Click on an instance from the Projects view to open its **Console**. The console is the command center for a single instance.

**Header**

- **Branch name** — the Git branch this instance tracks.
- **Stage badge** — Production (green), Staging (amber), or Dev (blue).
- **Status indicator** — a live colored dot showing the current instance state (running, stopped, error, etc.) with a text label.
- **Instance domain** — the full domain name shown below the branch name.
- **Open Instance** button — opens the Odoo URL in a new tab.

**Tab bar** — up to eight tabs depending on instance stage and member permissions.

### Quick Actions

A row of action buttons sits just below the header. Buttons are shown only when the current user has the corresponding permission.

| Action      | Color | Description                                        |
| ----------- | ----- | -------------------------------------------------- |
| **Restart** | Gray  | Restart the instance (see restart modes below)     |
| **Connect** | Gray  | Open the "Connect As User" session picker          |
| **Merge**   | Gray  | Merge another branch into this instance            |
| **Stop**    | Amber | Gracefully stop all containers                     |
| **Remove**  | Red   | Permanently delete the instance (Dev/Staging only) |

#### Restart Modes

Clicking **Restart** opens a modal with two modes:

| Mode             | What happens                                                                                                                                                                                                   |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Soft Restart** | Manually pulls codebase updates and restarts the Odoo service and background services, without restarting full container ( useful in rare cases such as manual code changes cause git conflicts in Web Editor) |
| **Hard Restart** | Stops and recreates the Docker containers from scratch. Use this when the Odoo process is unresponsive or after changing Docker-level or Nginx Settings.                                                       |

#### Connect As User

The **Connect** button opens the **Connect As User** modal. It lists all active Odoo sessions (users currently logged in) with their name and login. Use the search field to filter by name or login. Click **Connect** next to any user to open a new browser tab signed in as that user — useful for troubleshooting customer-reported issues without knowing their password.

#### Merge

Enter the **target branch** name in the merge modal and confirm. CICDoo will git-merge the current instance's branch into the specified target branch.

#### Remove Instance

Type `DELETE` in the confirmation input (case-sensitive) to irreversibly remove the instance and all its data. The **Remove** button is only visible on Dev and Staging instances — Production instances cannot be removed from the console.

### Commits Tab

Displays the most recent Git commits on the instance's branch in a table:

- Commit hash (abbreviated)
- Author
- Commit message
- Date / time ago

Click **Refresh** to reload the list.

### Access Tab

All credentials and connection details needed to access the instance are shown here.

> **Plan requirement:** IDE and remote session access require a Pro or Enterprise plan. A notice with an upgrade link is shown if these features are not available on the current plan.

**Instance URL**

The public Odoo URL. Click **Copy** to copy it to the clipboard.

**IDE Access** _(requires&#x20;_`perm_instance_editor`_&#x20;and&#x20;_`allow_ide`_)_

| Field        | Description                                                                        |
| ------------ | ---------------------------------------------------------------------------------- |
| IDE URL      | `https://<instance-domain>/ide` — click the external link icon to open it directly |
| IDE Password | Shown masked by default. Click the eye icon to reveal. Click **Copy** to copy.     |

**Database Connection**

| Field         | Description                                |
| ------------- | ------------------------------------------ |
| Database      | PostgreSQL database name for this instance |
| User          | Database user                              |
| Port          | Odoo application port                      |
| Postgres Port | Direct PostgreSQL port                     |

**SSH Tunnel**

A ready-to-paste SSH tunnel command that forwards the PostgreSQL port to your local machine:

```text
ssh -L <postgres_port>:localhost:<postgres_port> -N -f <user>@<instance_domain|host_IP_if_behind_cloudflare>
```

Copy this command and run it in a local terminal to connect a database client (e.g., DBeaver, pgAdmin, psql) directly to the instance database.

### Queue Tab

Shows recent jobs dispatched for this instance:

| Column              | Description                                             |
| ------------------- | ------------------------------------------------------- |
| Type                | Job type (deploy, restart, backup, merge, stop, remove) |
| Status              | queued / running / done / failed                        |
| Initiated by        | User who triggered the job                              |
| Created / Completed | Timestamps                                              |

Click the **log icon** on any row to view the full job output in a modal. Click **Refresh** to reload.

### Logs Tab

Real-time view of the Odoo application logs.

**Controls**

| Control          | Description                                            |
| ---------------- | ------------------------------------------------------ |
| **Stream**       | Open a live log stream in the terminal panel           |
| **Refresh**      | Reload the current log output                          |
| **Clear**        | Clear the display (does not delete logs on the server) |
| **Download**     | Download the current log as a `.txt` file              |
| **Auto-refresh** | Toggle automatic refreshing at a configurable interval |

The log panel uses a dark terminal-style display. The footer shows the total line count and the timestamp of the last update.

### Monitoring Tab

Real-time and historical resource usage charts for the instance containers.

**Controls**

| Control          | Description                              |
| ---------------- | ---------------------------------------- |
| **Refresh**      | Manually reload charts                   |
| **Auto-refresh** | Toggle 30-second automatic refresh       |
| **Time range**   | Select 6h, 24h, or 48h historical window |

**Charts**

Two chart groups are displayed:

1. **Odoo Container** — CPU usage (%) and memory usage (MB/GB) over time.
2. **Database Container** — CPU usage (%) and memory usage (MB/GB) over time.

Each chart supports zoom (scroll or pinch). Click any point on a chart to open the log viewer filtered to the corresponding timestamp. Loading and error states are handled gracefully.

### Backups Tab

Available on Production and Staging instances only (requires `perm_instance_backup` and `allow_backup`). See §9 for full details.

### Settings Tab — Odoo

The **Settings** tab contains six sub-tabs. Each sub-tab has its own **Save** button — changes are not applied until saved.

> **Plan requirement:** The Odoo and PostgreSQL sub-tabs require `allow_instance_tuning`. The Nginx sub-tab requires `allow_nginx_tuning`. When the plan does not include these, the panels are grayed out and non-interactive.

**Odoo sub-tab fields**

| Field                     | Default       | Description                                                                                                                                                              |
| ------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Shared Memory (MB)        | 64            | `--shm-size` for the Odoo container                                                                                                                                      |
| Memory Limit (MB)         | 0 (unlimited) | Docker memory limit for the Odoo container                                                                                                                               |
| Reserved Memory (MB)      | 0             | Docker memory reservation for the Odoo container                                                                                                                         |
| Odoo Configuration        | _(empty)_     | Free-text `odoo.conf` content appended to the generated config. Reserved keys (`admin_passwd`, `data_dir`, `*_port`) are managed automatically and will be ignored here. |
| Server Shared Addons      | Off           | Toggle on to mount a server-level shared addons directory into the container                                                                                             |
| Server Shared Addons Path | _(empty)_     | Absolute path on the server (shown when Server Shared Addons is on)                                                                                                      |
| External Filestore        | Off           | Toggle on to use a custom server path for the Odoo filestore instead of the default Docker volume                                                                        |
| External Filestore Path   | _(empty)_     | Absolute path on the server (shown when External Filestore is on)                                                                                                        |
| Backup Schedule           | _(empty)_     | Space-separated times in `HH:MM` format for automated backups (e.g., `02:00 14:00`). Leave empty to disable automatic backups.                                           |
| Auto Restart              | Off           | Automatically restart the instance if the Odoo process crashes                                                                                                           |

**Example&#x20;**`odoo.conf`**&#x20;snippet:**

```ini
workers = 4
max_cron_threads = 2
limit_time_cpu = 600
limit_time_real = 1200
```

Click **Save Odoo Configuration** to apply.

### Settings Tab — PostgreSQL

| Field                    | Default       | Description                                                                                                                 |
| ------------------------ | ------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Shared Memory (MB)       | 64            | `--shm-size` for the PostgreSQL container                                                                                   |
| Memory Limit (MB)        | 0 (unlimited) | Docker memory limit for the PostgreSQL container                                                                            |
| Reserved Memory (MB)     | 0             | Docker memory reservation                                                                                                   |
| PostgreSQL Extensions    | _(empty)_     | One extension per line to install (e.g., `postgis`, `pg_trgm`)                                                              |
| PostgreSQL Configuration | _(empty)_     | Free-text `postgresql.conf` content (e.g., `shared_buffers = 256MB`)                                                        |
| Anonymize Database       | Off           | When enabled, sensitive personal data is anonymized when this instance's database is cloned to a Staging or Dev environment |

Click **Save PostgreSQL Configuration** to apply.

### Settings Tab — Security

> **Plan requirement:** IP whitelisting requires `allow_ip_whitelist`.

| Field                | Description                                                                                                    |
| -------------------- | -------------------------------------------------------------------------------------------------------------- |
| CORS Origins         | One origin per line. Sets the `Access-Control-Allow-Origin` header.                                            |
| IP Whitelist         | CIDR ranges or individual IPs that are allowed to access the instance. One per line. Example: `192.168.1.0/24` |
| IP Blacklist         | IPs or ranges that are explicitly blocked. One per line.                                                       |
| Enable Rate Limiting | Toggle to activate Nginx-level rate limiting                                                                   |
| Login Rate Limit     | Rate limit for the login endpoint (e.g., `5r/m` = 5 requests per minute)                                       |
| General Rate Limit   | Rate limit for all other requests (e.g., `60r/s`)                                                              |
| Enable CSP           | Toggle to add a `Content-Security-Policy` response header                                                      |

Click **Save Security Settings** to apply.

### Settings Tab — Domain

| Field              | Description                                                                                                                         |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| Primary Domain     | Read-only display of the instance's current primary domain                                                                          |
| Extra Domains      | Additional custom domains that should route to this instance. One domain per line. DNS for each domain must point to the server IP. |
| Cloudflare Proxy   | Read-only indicator inherited from the server configuration                                                                         |
| Force Generate SSL | Toggle on to force SSL certificate regeneration on the next restart                                                                 |

> **DNS setup:** After adding an extra domain, create an `A` record in your DNS provider pointing the domain to the server's IP address. SSL certificates are issued automatically via Let's Encrypt on the next restart.

Click **Save Domain Settings** to apply.

### Settings Tab — Nginx

> **Plan requirement:** Requires `allow_nginx_tuning`.

| Field                 | Default    | Description                                                         |
| --------------------- | ---------- | ------------------------------------------------------------------- |
| Max Upload Size       | 100M       | `client_max_body_size` — maximum file upload size Nginx will accept |
| Proxy Read Timeout    | 300s       | Maximum time Nginx waits for the Odoo backend to respond            |
| Proxy Connect Timeout | 30s        | Maximum time Nginx waits to establish a connection to the backend   |
| Keepalive Timeout     | 65s        | How long keep-alive connections remain open                         |
| Send Timeout          | 60s        | Maximum time between successive write operations to the client      |
| X-Frame-Options       | SAMEORIGIN | HTTP header to control iframe embedding (`SAMEORIGIN` or `DENY`)    |

Click **Save Nginx Settings** to apply.

### Settings Tab — Packages

Add extra system packages to the Odoo or PostgreSQL container. Changing packages triggers a container rebuild on the next deploy.

| Field                      | Description                                                                                                       |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Odoo Debian Packages       | Extra `apt` packages installed in the Odoo container. One per line. Example: `libcups2-dev`, `libpq-dev`          |
| PostgreSQL Debian Packages | Extra `apt` packages installed in the PostgreSQL container. Example: `postgresql-contrib`, `postgresql-plpython3` |

> **Important:** Adding packages increases build time and image size. Only add packages your custom modules actually require.

Click **Save Package Settings** to apply.

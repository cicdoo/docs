---
title: 'Integrating Odoo with MSSQL #2'
deprecated: false
hidden: false
metadata:
  robots: index
---
This guide shows how to connect Odoo to a Microsoft SQL Server database from inside a CICDoo-managed instance, using the new **Container Startup Script** feature to install Microsoft's official ODBC driver at runtime — no custom Docker image required.

## The Challenge

Debian Bookworm's default APT repositories do not ship Microsoft's official MSSQL ODBC driver (`msodbcsql18`). The Python library `pyodbc` — the standard way to talk to SQL Server from Python — needs that driver plus a working unixODBC stack, neither of which is present in the stock Odoo container.

Until now, the workaround was either to maintain a custom Docker image with the driver baked in, or to use the platform's Debian packages free-text field — which only accepts package names from the default repos and so couldn't pull from Microsoft's repository.

With the new **Odoo Container Startup Script** field, you can add Microsoft's APT repo, install the official driver, and configure unixODBC — all in one place, all re-applied on every restart.

## The Solution: msodbcsql18 + Startup Script

Microsoft publishes the official ODBC driver for SQL Server (`msodbcsql18`) and the unixODBC development headers via their own APT repository. The startup script does three things in order:

1. Adds Microsoft's APT signing key and Bookworm repo.
2. Installs `msodbcsql18` and the supporting unixODBC packages.
3. Lets `apt` register the driver with `odbcinst` automatically (the package post-install script handles `/etc/odbcinst.ini`).

Because the script re-runs on every CICDoo restart, the driver is reapplied automatically whenever the container is recreated.

## Step 1 — Add Pyodbc to Your Project's `requirements.txt`

Python packages should not be installed from the startup script — they belong in your project's dependency file so they're tracked in version control and reproducible.

Add this line to your **project-level&#x20;**`requirements.txt` (or a module-level `requirements.txt` inside one of your addons):

```
pyodbc
```

CICDoo installs requirements automatically when the instance is restarted, so `pyodbc` will be available in Odoo's Python environment after the next restart.

## Step 2 — Add the Startup Script

Open your instance Console → **Settings** → **Odoo Configuration**, and paste the following into the **Startup Script (bash, runs as root in odoo container)** field:

```bash
#!/usr/bin/env bash
set -e

export DEBIAN_FRONTEND=noninteractive

# Add Microsoft's APT signing key (idempotent — overwrites if already present)
install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://packages.microsoft.com/keys/microsoft.asc \
    | gpg --dearmor --yes -o /etc/apt/keyrings/microsoft.gpg
chmod 0644 /etc/apt/keyrings/microsoft.gpg

# Register Microsoft's Debian Bookworm repo for prod packages
cat > /etc/apt/sources.list.d/mssql-release.list <<'REPO'
deb [arch=amd64,arm64 signed-by=/etc/apt/keyrings/microsoft.gpg] https://packages.microsoft.com/debian/12/prod bookworm main
REPO

apt-get update

# Install Microsoft's official ODBC driver + unixODBC stack.
# ACCEPT_EULA=Y is required by the msodbcsql18 package post-install.
ACCEPT_EULA=Y apt-get install -y --no-install-recommends \
    msodbcsql18 \
    unixodbc \
    unixodbc-dev

# Sanity check — driver should be registered automatically by the package
odbcinst -q -d
```

Click **Save Odoo Configuration**, then **Restart** the instance.

After the restart, check the restart log for:

```
Running startup script in odoo container as root...
...
[ODBC Driver 18 for SQL Server]
Odoo startup script executed successfully
```

That's it — the container is now ready to talk to MSSQL.

## Step 3 — Connect from Python

In any Odoo addon or `odoo shell`, you can now open a connection:

```python
import pyodbc

conn = pyodbc.connect(
    "DRIVER={ODBC Driver 18 for SQL Server};"
    "SERVER=your.mssql.host,1433;"
    "DATABASE=yourdb;"
    "UID=youruser;"
    "PWD=yourpassword;"
    "Encrypt=yes;"
    "TrustServerCertificate=no;"
)

cursor = conn.cursor()
cursor.execute("SELECT @@VERSION")
print(cursor.fetchone()[0])
```

### Connection-String Notes

- `DRIVER={ODBC Driver 18 for SQL Server}` — must match exactly the driver name registered by the `msodbcsql18` package. Run `odbcinst -q -d` inside the container to confirm.
- `SERVER=host,port` — Microsoft's driver uses a comma between host and port (not a colon, not `PORT=`). Default port is `1433`.
- **For named instances**, use `SERVER=host\\InstanceName`.
- **For Azure SQL**, use the FQDN (`yourserver.database.windows.net`) and keep `Encrypt=yes;TrustServerCertificate=no;`.
- `Encrypt=yes` is the default in Driver 18 and what Microsoft recommends. Set `TrustServerCertificate=yes` only for on-prem servers with self-signed certs you've decided to trust.

## Step 4 — Don't Hard-Code Credentials

Storing the MSSQL password in Python source is fine for a quick proof of concept, but for anything beyond that:

- **Use Odoo system parameters**: store the connection string in `ir.config_parameter` and read it at runtime (`self.env['ir.config_parameter'].sudo().get_param('mssql.dsn')`).
- **Use container environment variables**: add `MSSQL_HOST=...`, `MSSQL_USER=...`, `MSSQL_PASSWORD=...` to the **Environment Variables (odoo container)** field in Console settings, then read them via `os.environ` in your addon.
- **Never commit secrets** to the addons repo. The git history is hard to scrub later.

A typical pattern:

```python
import os, pyodbc

conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 18 for SQL Server}};"
    f"SERVER={os.environ['MSSQL_HOST']},{os.environ.get('MSSQL_PORT', '1433')};"
    f"DATABASE={os.environ['MSSQL_DB']};"
    f"UID={os.environ['MSSQL_USER']};PWD={os.environ['MSSQL_PASSWORD']};"
    f"Encrypt=yes;TrustServerCertificate=no;"
)
```

## Troubleshooting

### `Can't open lib 'ODBC Driver 18 for SQL Server' : file not found`

The driver name has to match the registered entry exactly — including spaces and case. Run `odbcinst -q -d` inside the container to see what's registered, and copy that string verbatim into your `DRIVER={...}` clause.

### `Login timeout expired` / `Unable to connect`

- Confirm the MSSQL server is reachable from the Odoo container's network: `docker exec <uuid> nc -zv your.mssql.host 1433`.
- Check firewall rules on the MSSQL server side.
- For Azure SQL, ensure the CICDoo server's IP is allowed in the Azure firewall settings.

### `SSL Provider: ... certificate verify failed`

Driver 18 enforces encrypted connections by default. For on-prem servers with a self-signed certificate, add `TrustServerCertificate=yes;` to the connection string. For production, install a proper certificate on the MSSQL server instead.

### Driver isn't listed by `odbcinst -q -d`

Re-check the script ran — look for `Odoo startup script executed successfully` in the restart log. If it failed, the line above it will explain why (most often: Microsoft's APT repo unreachable, or the GPG key URL changed).

### `pyodbc` import fails in Odoo

Make sure `pyodbc` is in your project's `requirements.txt` and that the instance has been restarted since you added it. CICDoo installs requirements during restart, not on every code change.

## Key Recommendations

- **Use Microsoft's official driver (**`msodbcsql18`**)** — it's the actively maintained, fully featured option and supports modern SQL Server (2012+) and Azure SQL out of the box.
- **Put&#x20;**`pyodbc`**&#x20;in&#x20;**`requirements.txt`, not in the startup script. Python deps belong with your code; OS-level packages belong in the startup script.
- **Make the startup script idempotent.** Everything above is — `apt-get install` is a no-op when packages are already installed, the keyring and repo file are simply rewritten each time.
- **Avoid hard-coding credentials.** Use environment variables or `ir.config_parameter`.
- **Restart once after editing the script.** Subsequent restarts will re-apply automatically — no rebuild needed.

## See Also

- [Microsoft ODBC Driver for SQL Server on Linux](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server)
- [pyodbc documentation](https://github.com/mkleehammer/pyodbc/wiki)

<br />

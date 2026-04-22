---
title: Monitoring & Logs
deprecated: false
hidden: false
metadata:
  robots: index
---
## &#x20;

### Instance-Level Monitoring

The **Monitoring** tab in the Console provides per-instance CPU and memory charts for both the Odoo and database containers. Charts support zoom and time range selection (6h / 24h / 48h). Clicking a chart point opens the Logs tab at the corresponding timestamp.

### Server-Level Monitoring

From the Servers page, click the **charts icon** on a server card to open the server charts modal. This shows host-level resource metrics (CPU, memory, disk I/O) for the server machine.

### Deployment Logs

Every job (deploy, restart, merge, backup, stop, remove) produces a detailed log accessible from:

- The **Recent Deployments** widget on the Dashboard.
- The **Queue tab** inside the Console.
- The global **Queue** page.

### Live Log Streaming

The **Logs tab** in the Console provides a real-time terminal-style view of the running Odoo application log. Auto-refresh keeps the view current at a configurable interval. Logs can be downloaded as `.txt` files.

### Server Monitoring Agent

The `odoobot_monitor` service installed by the server preparation script reports the following metrics to CICDoo every 5 minutes:

- CPU usage and core count
- RAM and swap usage
- Disk space and inode usage
- Per-container CPU and memory stats
- Container lifecycle events

This data powers both the server charts and the instance monitoring charts.

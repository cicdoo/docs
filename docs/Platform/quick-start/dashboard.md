---
title: Dashboard
deprecated: false
hidden: false
metadata:
  robots: index
---
## Dashboard

The dashboard is the first screen you see after logging in. It provides a live overview of your platform.

![](https://files.readme.io/3511e3eef356e81f3a8b516b07c36228f15cb4fe28a693c8692af134b45fbffb-image.png)

### Stats Grid

Four summary tiles at the top of the page show:

| Tile             | What it counts                    |
| ---------------- | --------------------------------- |
| Servers          | Total registered servers          |
| Projects         | Total projects under your account |
| Active Instances | Running Odoo environments         |
| Deployments      | Total deployments processed       |

Metrics are color-coded to indicate healthy, warning, or critical states.

### Plan Usage Banner

Displays your current plan (e.g., Free) and quota consumption (servers used / limit, projects used / limit). An **Upgrade** link is shown when you are near or at a limit.

### Recent Deployments

![](https://files.readme.io/40e8fb9eb1e420bcc0cc32d763a2f851bea9f98ce86936e1ee3a6b7ee9863080-image.png)

A live-updating table showing the last five queue jobs. The table refreshes automatically every five seconds. Columns include:

- Server name
- Instance name
- Job type (e.g., deploy, restart, backup)
- Status (queued, running, done, failed)
- Initiated by (user name)
- Created time

Click the **log icon** on any row to open the full deployment log in a modal window.

### Quick Actions


<Image src="https://files.readme.io/7d59e9181ac01844e7c83c504f64882f4eca5f0979f4fa04612899a561d34961-image.png" align="center" width="50%" border={true} />


Shortcut buttons for the most common tasks, accessible directly from the dashboard without navigating to a specific resource.

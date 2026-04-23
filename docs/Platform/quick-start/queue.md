---
title: Queue
deprecated: false
hidden: false
metadata:
  robots: index
---
The **Queue** page shows every job that has been dispatched across all your instances.

**Columns**

| Column       | Description                                             |
| ------------ | ------------------------------------------------------- |
| Server       | Which server the job ran on                             |
| Instance     | Which instance was targeted                             |
| Type         | Job type (deploy, restart, backup, merge, stop, remove) |
| Status       | Current state: queued / running / done / failed         |
| Initiated by | User who triggered the job                              |
| Created      | Timestamp                                               |

**Log Viewer**

Click the log icon on any row to open the queue log modal, which shows the full console output of the job — including git pull output, module update logs, container start/stop events, and any errors.

**Pagination**

Use the page controls at the bottom to navigate through older jobs.

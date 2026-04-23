---
title: Backups & Restore
deprecated: false
hidden: false
metadata:
  robots: index
---
Backups are managed from the **Backups tab** inside the Console of a Production or Staging instance.

> **Plan requirement:** The backup feature requires `allow_backup`. On plans that do not include it, a plan restriction message is shown with an upgrade link.

### Creating a Backup

Click **Create Backup** to open the backup creation modal. Choose one of two backup scopes:

| Scope                    | What is included                                                 |
| ------------------------ | ---------------------------------------------------------------- |
| **Database only**        | PostgreSQL dump of the instance database                         |
| **Database + Filestore** | Database dump AND all uploaded attachments/documents (filestore) |

Click **Create** to queue the backup job. The backup appears in the list once the job completes. A full backup with a large filestore may take several minutes.

### Backup List

Each backup entry shows:

- Backup type (database only or full)
- File size
- Time since creation (e.g., "3 hours ago")

### Downloading a Backup

Click the **Download** button next to any backup to download the archive to your local machine.

### Restoring a Backup

Click **Restore** next to the backup you want to apply. A confirmation prompt is shown. Restoring replaces the current database (and filestore, for full backups) with the backup content.

> **Warning:** Restore is a destructive operation. The current database state will be permanently overwritten. Always ensure you have a recent backup before restoring.

### Deleting a Backup

Click **Delete** next to a backup. Confirm in the dialog. Deleted backups cannot be recovered.

### Automated Backups

Backups can be scheduled to run automatically from the instance's **Settings → Odoo** sub-tab. Set the **Backup Schedule** field to one or more times in `HH:MM` format (space-separated). For example, `02:00 14:00` schedules backups at 2 AM and 2 PM daily.

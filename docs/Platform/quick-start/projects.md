---
title: Projects
deprecated: false
hidden: false
metadata:
  robots: index
---
A project maps to a Git repository and an Odoo configuration. Each project can have up to three environments: Production, Staging, and Development, each deployed on its own server.

### Creating a Project

Click **New Project** to open the project creation form.

**Step 1 — Git Provider**

Select **GitHub** or **GitLab**. The form will check whether you have authorized CICDoo to access repositories on the selected provider.

**Step 2 — Project Type**

| Type                | Description                                                |
| ------------------- | ---------------------------------------------------------- |
| New project         | CICDoo creates a fresh repository for you                  |
| Existing repository | Connect an existing repo from your account or organization |

**Step 3 — Repository Search**

Type in the search box to find repositories by name. Results are fetched live from your connected Git provider. Select the desired repository from the dropdown.

**Step 4 — Project Details**

| Field        | Description                                       |
| ------------ | ------------------------------------------------- |
| Project name | Human-readable name shown in the interface        |
| Odoo version | 11.0 through 19.0, or `master`                    |
| Edition      | Community Edition (CE) or Enterprise Edition (EE) |

Click **Create Project** to finish. The project appears in the Projects grid.

### Connecting a Git Repository

If you have not yet authorized GitHub or GitLab, the form will show an authorization banner.

**OAuth Authorization** — click **Authorize with GitHub / GitLab** to grant repository access via the standard OAuth flow. Recommended for most users.

**Personal Access Token (PAT)** — if OAuth is not available (e.g., GitLab self-hosted), expand the PAT section and enter:

- For **GitHub**: a fine-grained or classic PAT with `repo` scope.
- For **GitLab**: a PAT with `api` and `read_repository` scopes.

Save the PAT; it is stored encrypted and used for all subsequent Git operations on your behalf.

### Project Settings — General Tab

Navigate to a project and click the **Settings** button (or the gear icon) to open Project Settings. The **General** tab is shown by default.

| Field            | Description                                                                                                 |
| ---------------- | ----------------------------------------------------------------------------------------------------------- |
| **Project UUID** | Read-only unique identifier for API and script use. Click **Copy** to copy it to the clipboard.             |
| **Odoo Version** | The target Odoo version (11.0–19.0 or `master`). Changing this affects new deploys.                         |
| **Edition**      | Community Edition (`ce`) or Enterprise Edition (`ee`). Switching to EE requires an `allow_ee_edition` plan. |
| **Time Machine** | Toggle to pin instances to exact Odoo commit hashes. See §5.8.                                              |

Click **Save Changes** to apply updates.

### Project Settings — GitHub Tab

Displays the connected repository information and lets you update the production branch.

**Repository Card**

Shows the repository name, a link to view it on GitHub, and (if available) the star count, fork count, and repository description.

**Default Branch**

| Field          | Description                                                     |
| -------------- | --------------------------------------------------------------- |
| Default Branch | The branch used as the production instance. Defaults to `main`. |

> **Note:** Changing the Default Branch here does not automatically redeploy the production instance — it changes which branch is tracked on the next deploy.

### Project Settings — Environments Tab

Assigns each environment (Production, Staging, Development) to a server. Each section shows a DNS validation badge when the domain is confirmed.

**Per-environment fields**

| Field               | Description                                                         |
| ------------------- | ------------------------------------------------------------------- |
| Server search       | Searchable dropdown of all registered servers. Type to filter.      |
| Current server      | The currently assigned server hostname is shown below the dropdown. |
| DNS Validated badge | Green badge shown when DNS is correctly pointed.                    |

The three environments are:

- **Production** (green star icon) — live customer-facing instance; Remove action is disabled.
- **Staging** (amber flask icon) — pre-production testing environment.
- **Development** (blue code icon) — daily development branch.

To assign a server, click the search field, type part of the server hostname or IP, and select from the list. Click **Save Changes** to apply.

### Project Settings — Storage Tab

Configure S3-compatible object storage for all instances in the project. Backups and filestores can be stored on AWS S3, MinIO, DigitalOcean Spaces, Wasabi, Backblaze B2, or any S3-compatible provider.

**Basic Settings**

| Field             | Description                                                               |
| ----------------- | ------------------------------------------------------------------------- |
| S3 Bucket Name    | The target bucket for this project (e.g., `my-project-bucket`)            |
| Region / Location | AWS region (e.g., `us-east-1`, `eu-west-1`) or `auto` for other providers |
| Access Key ID     | Your S3 access key ID (e.g., `AKIAIOSFODNN7EXAMPLE`)                      |
| Secret Access Key | Your S3 secret key (stored encrypted)                                     |

**S3-Compatible Provider Configuration**

| Field               | Description                                                                                                                             |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Custom Endpoint URL | Required for non-AWS providers (e.g., `https://nyc3.digitaloceanspaces.com`, `https://minio.example.com:9000`). Leave empty for AWS S3. |
| Use SSL/TLS (HTTPS) | Toggle on to enforce HTTPS connections (recommended).                                                                                   |

**S3 Operations**

Three action buttons are available after filling in the credentials:

| Button              | Action                                                                               |
| ------------------- | ------------------------------------------------------------------------------------ |
| **Test Connection** | Verifies your credentials and endpoint by attempting to list the bucket contents.    |
| **Create Bucket**   | Creates the configured bucket in your S3 provider.                                   |
| **Upload Files**    | Opens the upload modal to transfer files directly to the bucket with a progress bar. |

> **Best practice:** Click **Test Connection** before saving to confirm credentials are correct. The upload modal shows a real-time progress bar and allows cancellation mid-upload.

### Project Settings — Advanced Tab

#### IDE Extensions

```text
IDE Extensions textarea (monospaced)
```

Enter the VS Code extension identifiers you want pre-installed in the web IDE for all instances of this project. One extension ID per line, for example:

```text
ms-python.python
ms-python.vscode-pylance
odoo-samples.odoo-snippets
```

Extensions are installed when an instance is (re)built. See §7 for more about the web IDE.

#### Extra Repositories

```text
Extra Repositories textarea (monospaced)
```

List additional Git repositories that should be cloned alongside the main project. Each line has the format:

```text
Repository URL, Branch Name, Commit Hash
```

Example:

```text
https://github.com/OCA/web.git, 17.0,
https://github.com/OCA/server-tools.git, 17.0, a1b2c3d4
```

- **Repository URL** — full HTTPS clone URL.
- **Branch Name** — branch to clone.
- **Commit Hash** — optional. If provided, the repo is checked out at that exact commit (pin behavior).

These repositories are cloned into the instance's addons path alongside your main repository, making their modules available to Odoo automatically.

#### Danger Zone

The **Delete Project** button permanently removes the project and **all associated instances**. There is no undo. A confirmation dialog is shown before proceeding.

### Time Machine

Time Machine lets you pin Odoo source code to exact git commit hashes, giving you full reproducibility over the Odoo version deployed.

**Enable Time Machine**

Toggle **Time Machine** on in the General tab. Three commit hash fields appear:

| Field                  | Required | Description                                                    |
| ---------------------- | -------- | -------------------------------------------------------------- |
| Community Commit Hash  | Always   | Commit SHA for Odoo Community source                           |
| Enterprise Commit Hash | EE only  | Commit SHA for Odoo Enterprise source (hidden for CE projects) |
| Themes Commit Hash     | Always   | Commit SHA for the official themes repository                  |

All required fields must be filled before saving. Instances built under this project will use exactly those commits, regardless of upstream branch updates.

> **Use case:** Pin to a known-good commit before a go-live freeze, or reproduce a historical environment for debugging a version-specific bug.

### Deleting a Project

Go to **Project Settings → Advanced → Danger Zone** and click **Delete Project**. Confirm in the dialog. This action:

- Removes the project record and all metadata.
- Removes **all associated instances** (Production, Staging, Development) and their databases.
- Cannot be undone.

> Back up all databases before deleting a project if you need to preserve data.

<br />

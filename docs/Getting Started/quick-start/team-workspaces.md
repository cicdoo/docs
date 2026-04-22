---
title: Team & Workspaces
deprecated: false
hidden: false
metadata:
  robots: index
---
Workspaces let multiple team members share access to a set of servers, projects, and instances with fine-grained permission control.

### Creating a Workspace

1. Go to **Settings → Workspaces**.
2. Click **Create Workspace**.
3. Enter a workspace **name**. A URL-safe slug is generated automatically as you type.
4. Click **Create**.

The workspace gets its own URL prefix (`/<workspace-slug>/...`). Switching workspaces changes the navigation context for all pages.

### Inviting Members

Inside a workspace, click **Invite Member** and enter the invitee's **email address**. They receive an invitation email with a link to accept. Once accepted, they appear in the member list with default (no) permissions.

### Member Permissions

Each member has an independent permission profile. Click the **permissions icon** next to a member to open the permissions modal.

Permissions are divided into two sections:

#### Action Permissions

Toggle each permission independently:

| Member Permissions                                   |
| ---------------------------------------------------- |
| Access the Projects section and view project details |
| Trigger soft or hard restarts                        |
| Open the "Connect As User" session picker            |
| Merge branches into target branches                  |
| Stop running instances                               |
| Permanently delete Dev/Staging instances             |
| Create, download, restore, and delete backups        |
| View application and deployment logs                 |
| Access the web-based IDE                             |
| Open and modify the instance Settings tab            |

#### Resource Access

Restrict which specific resources the member can see and interact with:

- **Allowed Projects** — leave empty = no project access; add projects = access only to those.
- **Allowed Instances** — leave empty = no instance access; add instances = access only to those.
- **Allowed Servers** — leave empty = no server access; add servers = access only to those.

> **Note:** Workspace owners bypass all permission checks and have full access to every resource regardless of these settings.

### Workspace Customization

From **Settings → Workspaces**, you can customize the workspace appearance:

**Logo**

- Click the logo area or the **Upload** button to select an image file.
- Click **Remove** to revert to the default logo.
- Uploaded logos are displayed in the sidebar and header.

**Primary Color**

- A color swatch picker lets you choose a brand color applied across the workspace UI.
- Select a predefined swatch or enter a hex code directly.

### AI Settings

Each workspace can be configured with its own AI provider for AI-assisted features within the platform.

| Field    | Description                                                               |
| -------- | ------------------------------------------------------------------------- |
| Provider | Anthropic, OpenAI, Groq, Ollama, or Custom                                |
| Model    | The specific model to use (e.g., `claude-sonnet-4-6`, `gpt-4o`, `llama3`) |
| API Key  | Your API key for the chosen provider (stored encrypted)                   |
| Base URL | Required for Ollama and Custom providers (e.g., `http://localhost:11434`) |

<br />

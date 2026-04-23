---
title: Account Settings
deprecated: false
hidden: false
metadata:
  robots: index
---
Access the Settings page from the user menu or sidebar. Five tabs cover different areas.

### Profile

Update your personal information:

- **Name**
- **Email address**
- **Avatar / profile picture**

### Security

- **Change password** — enter your current password and a new one.
- **Two-Factor Authentication (2FA)** — enable TOTP-based 2FA for additional login security. Scan the QR code with an authenticator app (Google Authenticator, Authy, etc.).
- **Active sessions** — view and revoke active browser sessions from other devices.

### Integrations (GitHub & GitLab)

Displays the current connection status for each Git provider.

**GitHub**

- If not connected: **Authorize with GitHub** button initiates the OAuth flow.
- **Personal Access Token (PAT)**: expand to enter a classic or fine-grained PAT as an alternative to OAuth. Required scopes: `repo`.

**GitLab**

- Same flow as GitHub.
- PAT required scopes: `api`, `read_repository`.

Once connected, the authorization status and connected account are displayed. You can disconnect and reconnect at any time.

### Billing

**Current Plan**

Shows your active plan name and billing interval (monthly or yearly).

**Usage Statistics**

| Metric                | Display   |
| --------------------- | --------- |
| Projects used         | X / limit |
| Servers used          | X / limit |
| Instances per project | X / limit |
| Workspace members     | X / limit |

**Feature Checklist**

A visual indicator for each plan-gated feature:

- Private repositories
- Automated backups
- Web IDE access
- Branch merging
- Enterprise Edition (EE) support
- Instance performance tuning
- Nginx tuning
- Team workspace

**Upgrading**

Click **Upgrade** to open the billing interval selection modal. Choose **Monthly** or **Yearly** (yearly pricing shows the percentage savings). Confirm to proceed to the payment flow.

**Subscription Status**

- If cancellation is scheduled, a notice shows the date access ends.
- If a payment is past due, an alert prompts you to update your payment method.

<br />

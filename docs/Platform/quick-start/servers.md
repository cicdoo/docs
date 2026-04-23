---
title: Servers
deprecated: false
hidden: false
metadata:
  robots: index
---
## Servers

Servers are the Linux machines that host your Odoo instances. You must register at least one server before creating instances.

### Adding a Server

Click the **Add Server** button to open the new-server form.

**Domain / IP Configuration**

| Option              | When to use                                                         |
| ------------------- | ------------------------------------------------------------------- |
| Use a CICDoo domain | CICDoo manages the subdomain for you (e.g., `timestamp.cicdoo.com`) |
| Use own domain / IP | You provide a custom domain name or bare IP address                 |

When using your own domain, fill in:

- **Domain or IP** — the hostname or IP that CICDoo will connect to.
- **Connectivity IP override** — an alternative IP used only for initial SSH connectivity (useful when the domain resolves differently externally vs. internally).

**Username**

The Linux user CICDoo will use to SSH into the server. This user must have permission to run Docker commands (typically a member of the `docker` group).

### Server Authentication Methods

Choose one of three SSH authentication methods:

| Method          | How it works                                                                                                                                                       |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Password**    | Enter the user's password. Stored encrypted.                                                                                                                       |
| **Private Key** | Paste an existing private key (RSA, Ed25519). CICDoo stores it encrypted and uses it for all SSH operations.                                                       |
| **Public Key**  | CICDoo generates a key pair. You copy the displayed public key and add it to `~/.ssh/authorized_keys` on the server. A link to the SSH setup tutorial is provided. |

> **Tip:** The public-key method is the most secure option. Use the one-click **Copy** button to copy the public key, then follow the tutorial to install it on your server.

### Cloudflare & Load Balancer Options

**Cloudflare Proxy**<br />Toggle on if your domain is proxied through Cloudflare. CICDoo will adjust its SSL handling to work correctly behind Cloudflare's proxy layer.

**Load Balancer**<br />Toggle on if the server sits behind a load balancer. When enabled, provide the **load balancer IP** so that CICDoo can configure upstream rules correctly.

### Editing a Server

From the Servers page, click the **edit icon** on any server card to open the server edit modal. All fields set during creation can be updated, including credentials and domain settings.

### Server Logs & Charts

From any server card you can access:

- **Logs** — raw system-level logs from the server agent.
- **Charts** — historical CPU and memory utilization for the server host.

### Server Preparation&#x20;

Before CICDoo can deploy instances on a server, the server must be prepared with Docker, Nginx, and the CICDoo monitoring agent. CICDoo provides an automated setup  for Ubuntu servers (20.04, 22.04, 24.04).

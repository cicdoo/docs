---
title: Code Pushed but Not Reflected in Odoo Container
deprecated: false
hidden: false
metadata:
  robots: index
---
## 1. 🔍 Verify the Commit Exists

Check your Git history (locally or in your Git provider):

### ❌ Commit NOT found

- You are likely pushing to the wrong repository
- Or using the wrong upstream/remote

**Fix:**
git remote -v
git branch -vv

***

### ✅ Commit IS found

→ Proceed to the next step

***

## 2. 📦 Check Deployment Queue

Go to your CICDoo dashboard → **Queue tab**

Check if an **upgrade process** was triggered:

- ⏳ **Queued / Processing**<br />Deployment is in progress → wait
- ❌ **Failed**<br />Check logs → identify the issue (dependency, migration, syntax, etc.) or contact support
- ✅ **Done**<br />Deployment completed, but code still not updated?
  - Possible caching issue
  - Wrong branch
  - Module not reloaded ( contact support )

***

## 3. ⚠️ No Upgrade Process Triggered?

If **no deployment was triggered after push**, check:

### 🔗 GitHub Webhooks

Go to your repository → **Settings → Webhooks**

Verify:

- Webhook exists
- URL points to:
  app.cicdoo.com

### ❌ Issues

- No webhook ( contact support )
- Incorrect webhook URL

**Fix:**

- Add or correct the webhook
- Or contact support

***

## 4. 💳 Subscription / Billing Check

Even if everything looks correct:

### ❌ Possible Issue

- Subscription suspended
- Outstanding payments

**Impact:**

- Deployments may be blocked silently

**Fix:**

- Ensure account is active
- Resolve billing issues

***

## 5. 🧠 Common Gotchas

- Wrong branch configured in CICDoo (e.g., pushing to `dev` while app tracks `main`)
- Module not updated (`-u module_name` not triggered)
- Odoo cache not refreshed
- Confusion between multiple repositories (fork vs origin)

***

## 🧾 Quick Decision Flow

```text
Commit missing?
  → Wrong repo / upstream

Commit exists?
  → Check queue

Queue empty?
  → Check webhook

Webhook OK?
  → Check subscription

All OK?
  → Check logs / branch / module update
```

<br />

# SRE Ticket: Set up Private GitHub App for Autonomous SRE Operations

This runbook guides you through replacing your personal fine-grained Personal Access Token (PAT) with a private **GitHub App** integration. 

Using a GitHub App provides enterprise-grade security for our autonomous SRE watchdogs:
1.  **Auto-Rotating Keys:** Eliminates static, long-lived tokens in our local file system.
2.  **Explicit Scope Isolation:** Locks down our bot strictly to the required repository actions.
3.  **Clean Audit Trail:** All automated SRE actions will show up under your bot's app name (e.g., `fkcurrie-k3s-sre-bot [bot]`) rather than your personal GitHub account.

---

## 🛠️ Step 1: Create the Private GitHub App

You can create this private app directly under your personal GitHub account:

1.  Log into **GitHub** ➡️ Go to your profile **Settings** (top-right dropdown) ➡️ Scroll to the bottom of the left sidebar and click **Developer Settings** ➡️ **GitHub Apps** ➡️ Click **New GitHub App**.
2.  **App Name:** `fkcurrie-k3s-sre-bot` (or any unique name of your choice).
3.  **Homepage URL:** Enter your repository URL: `https://github.com/fkcurrie/k3s-gitops`.
4.  **Webhooks:** Uncheck the box for **"Active"** (we use cron-based polling, so webhooks are not needed).
5.  **Repository Permissions:** Expand the "Repository permissions" accordion and configure these exact scopes:
    *   📁 **Contents:** `Read and write` (For pushing config updates, branching, and PR fixes)
    *   🔌 **Issues:** `Read and write` (For logging SRE alerts and notifications)
    *   🔀 **Pull Requests:** `Read and write` (For SRE automated review and command execution)
    *   ⚙️ **Workflows:** `Read and write` (To allow the push and execution of our YAML/Kubernetes linter)
6.  **Where can this GitHub App be installed?:** Select **"Only on this account"** (keeps the App private to your personal repositories).
7.  Click **Create GitHub App**.

---

## 🔑 Step 2: Extract Credentials & Install

1.  **Copy App ID:** On the general App Settings page, locate and copy the numerical **App ID** (e.g., `123456`).
2.  **Generate Private Key:** Scroll down to the bottom of the page and click **"Generate a private key"**. This will download a `.pem` file to your computer (e.g., `fkcurrie-k3s-sre-bot.2026-05-31.private-key.pem`).
3.  **Install the App on your Repo:**
    *   On the left menu, click **Install App**.
    *   Click **Install** next to your personal GitHub organization/account.
    *   Select **"Only select repositories"** and choose your **`k3s-gitops`** repository.
    *   Click **Install**.

---

## 🚀 Step 3: Deliver to Hermes

Once you have completed steps 1 & 2, simply paste the following details back to Hermes in our Discord chat:

1.  The numerical **App ID**.
2.  The full text/content of your downloaded `.pem` **Private Key file**.

Hermes will automatically:
*   Securely store the private key locally in the cluster (outside your Git directories).
*   Transition our 2-minute and 15-minute SRE cron jobs to use dynamically generated, hourly expiring installation tokens.
*   Commit and push our automated `.github/workflows/lint.yaml` linter, securing your GitOps flow!

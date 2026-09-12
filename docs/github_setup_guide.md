# GitHub Integration & Webhook Setup Guide

This comprehensive guide walks you through connecting a GitHub repository to the **Commit Overflow Backend**. Once connected, Commit Overflow automatically tracks issues, pull requests, reviews, comments, and awards points and streaks to contributors.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Backend Configuration (`.env`)](#2-backend-configuration-env)
3. [Registering the Repository in Commit Overflow](#3-registering-the-repository-in-commit-overflow)
4. [Configuring the Webhook on GitHub](#4-configuring-the-webhook-on-github)
5. [Configuring GitHub Labels & Workflow](#5-configuring-github-labels--workflow)
6. [Testing the Connection](#6-testing-the-connection)
7. [Troubleshooting & Common Issues](#7-troubleshooting--common-issues)

---

## 1. Prerequisites

Before setting up the webhook, ensure:
- **Publicly Accessible Backend URL**: GitHub servers need to send HTTP `POST` requests to your backend.
  - **Production**: A domain with valid HTTPS (e.g., `https://api.commitoverflow.com`).
  - **Local Development**: Use a tunneling tool such as [ngrok](https://ngrok.com/), [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/), or [localtunnel](https://localtunnel.github.io/www/) (e.g., `https://abc123.ngrok-free.app`).
- **Event Status is Active**: The backend will ignore all incoming webhooks if the event state is not `ACTIVE`.
  - In Admin Dashboard: Set Event Status to **Active**.
  - Or via SQL:
    ```sql
    UPDATE event_state SET status = 'ACTIVE' WHERE key = 'singleton';
    ```
- **Admin Access**:
  - Admin access to the Commit Overflow Admin Panel.
  - Administrator/Maintainer permissions on the target GitHub repository.

---

## 2. Backend Configuration (`.env`)

Open your backend `.env` file (in `Commit-overflow-backend/`) and configure the following variables:

```env
# ──────────────────────────────────────────────────────────────────────────────
# GitHub Configuration
# ──────────────────────────────────────────────────────────────────────────────

# Required: Secret used to verify payload authenticity via HMAC-SHA256
GITHUB_WEBHOOK_SECRET=your_strong_random_secret_here

# Required for API sync: Personal Access Token (classic or fine-grained)
GITHUB_TOKEN=ghp_yourPersonalAccessTokenHere
```

### Generating the Secrets & Tokens:
1. **`GITHUB_WEBHOOK_SECRET`**:
   - Generate a strong random string (e.g., run `openssl rand -hex 20` in your terminal or use a password generator).
   - Save this value—you will use the exact same string when configuring the webhook in GitHub.
2. **`GITHUB_TOKEN`**:
   - Go to GitHub -> **Settings** -> **Developer Settings** -> **Personal access tokens** -> **Tokens (classic)**.
   - Click **Generate new token (classic)**.
   - Give it a descriptive note (e.g., `Commit Overflow Sync`).
   - Select scopes:
     - `repo` (Full control of private and public repositories, issues, pull requests, and commit statuses).
     - `read:user`, `user:email` (to validate and match user profiles).
   - Generate the token and paste it into `GITHUB_TOKEN`.

Restart the backend after updating `.env`.

---

## 3. Registering the Repository in Commit Overflow

The backend strictly filters incoming webhooks by comparing the repository URL received in the payload (`repository.html_url`) with the `github_url` registered in the database.

### Option A: Using the Admin Dashboard
1. Log in as an Admin in the Commit Overflow web app.
2. Navigate to **Admin Panel** -> **Repositories**.
3. Click **Add Repository**.
4. Fill in:
   - **Repository Name**: e.g., `my-awesome-project`
   - **GitHub URL**: `https://github.com/<owner>/<repo>` *(Do not include trailing slashes or `.git`)*.
   - **Tech Stack**: Select the applicable tags (e.g., `go`, `react`, `typescript`).
   - **Active**: Checked (`true`).
   - **Integrated**: Checked (`true`).
5. Click **Save**.

### Option B: Via Backend API (Admin)
```http
POST /admin/repos
Authorization: Bearer <ADMIN_JWT_TOKEN>
Content-Type: application/json

{
  "name": "my-awesome-project",
  "github_url": "https://github.com/organization/repo-name",
  "tech_stack": ["golang", "vue"],
  "is_active": true,
  "is_integrated": true
}
```

---

## 4. Configuring the Webhook on GitHub

1. Open your repository on GitHub.
2. Navigate to **Settings** -> **Webhooks** (in the left sidebar).
3. Click **Add webhook** (you may be prompted to enter your GitHub password/2FA).
4. Configure the webhook settings:

| Field | Value | Notes |
|---|---|---|
| **Payload URL** | `https://<YOUR_BACKEND_DOMAIN>/webhooks/github` | E.g. `https://api.commitoverflow.com/webhooks/github` |
| **Content type** | `application/json` | **Crucial!** Do not use `application/x-www-form-urlencoded` |
| **Secret** | The exact value of `GITHUB_WEBHOOK_SECRET` | Matches the secret in your `.env` file |
| **SSL verification** | **Enable SSL verification** | Recommended for production |

5. Under **Which events would you like to trigger this webhook?**:
   - Choose **Let me select individual events**.
   - Check the following events:
     - [x] **Issues** *(Opened, edited, deleted, transferred, pinned, closed, reopened, assigned, unassigned, labeled, unlabeled)*
     - [x] **Issue comments** *(Created, edited, deleted)*
     - [x] **Pull requests** *(Opened, closed, reopened, edited, assigned, unassigned, review requested, synchronize)*
     - [x] **Pull request reviews** *(Submitted, edited, dismissed)*
6. Ensure **Active** is checked.
7. Click **Add webhook**.

---

## 5. Configuring GitHub Labels & Workflow

Commit Overflow calculates points, deadlines, and issue approval automatically based on repository labels configured in your `label_configs` table:

1. **Difficulty Labels**:
   - Default configured labels typically include: `easy`, `medium`, `hard` (or `level1`, `level2`, etc.).
   - Create corresponding labels in your GitHub repository (**Issues** -> **Labels** -> **New label**).
   - When an issue is given one of these labels, Commit Overflow transitions the issue to `approved`, sets the deadline based on `duration_hours`, and calculates points.
2. **Special Administrative Labels**:
   - `invalid`: Marks the issue as invalid and schedules it for auto-closing in 6 hours.
   - `duplicate`: Marks the issue as duplicate and schedules it for auto-closing in 6 hours.
3. **Assignees**:
   - When an issue is assigned to a contributor on GitHub, Commit Overflow sets the assignee, transitions the issue to `assigned`, and starts their countdown timer.
   - If reassigned, the previous assignee may receive a penalty deduction as configured.

---

## 6. Testing the Connection

1. **Check the Webhook Ping**:
   - In GitHub -> **Settings** -> **Webhooks**, click on your webhook.
   - Scroll down to the **Recent Deliveries** section.
   - You should see a `ping` event with a green checkmark (`200 OK`).
   - If it shows `200 OK` with response `{"status":"event ignored","event":"ping"}`, the connection and signature verification succeeded!
2. **Test an Issue Event**:
   - Create a test issue in your GitHub repository.
   - Go to Commit Overflow -> **Issues** or **Admin Panel** -> **Issues**.
   - The issue should appear in the system under `pending` state.
   - Check the backend logs:
     ```
     [WEBHOOK] issue opened - github_user=username repo=repo-name issue=#1
     ```
3. **Test Label Sync**:
   - Add a difficulty label (e.g., `medium`) to the issue on GitHub.
   - Verify the issue status updates to `approved` in Commit Overflow.

---

## 7. Troubleshooting & Common Issues

| Problem | Cause | Solution |
|---|---|---|
| **HTTP 401 Unauthorized** | The HMAC SHA-256 signature did not match. | Verify that `GITHUB_WEBHOOK_SECRET` in `.env` is identical to the **Secret** entered in GitHub Webhook settings. Ensure the backend was restarted after updating `.env`. |
| **HTTP 400 Bad Request** | The payload could not be parsed. | Ensure **Content type** in GitHub is set to `application/json`, not `application/x-www-form-urlencoded`. |
| **Response: `ignored: event is not active`** | The event status is paused or inactive. | In Admin Panel, toggle the Event to **Active**, or run `UPDATE event_state SET status = 'ACTIVE' WHERE key = 'singleton';`. |
| **Response: `repo not tracked`** | The repository URL doesn't match the database. | Check the registered `github_url` in the database. It must match `https://github.com/<owner>/<repo>` exactly without trailing slashes. |
| **Delivery Failed / Connection Timed Out** | GitHub cannot reach your backend. | Ensure your backend is public or your tunnel (e.g. ngrok) is running and the URL in GitHub Webhook settings matches the tunnel URL. |
| **Points not awarded on merge** | PR was not linked to an approved issue or user not mapped. | Ensure the PR description references the issue (e.g., `Fixes #12`) and the contributor has signed in with their GitHub account. |

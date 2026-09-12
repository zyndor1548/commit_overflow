# GitLab Integration & Webhook Setup Guide

This comprehensive guide walks you through connecting a GitLab repository to the **Commit Overflow Backend**. Once connected, Commit Overflow automatically tracks issues, merge requests (MRs), assignees, label changes, and awards points and streaks to contributors.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Backend Configuration (`.env`)](#2-backend-configuration-env)
3. [Registering the Repository in Commit Overflow](#3-registering-the-repository-in-commit-overflow)
4. [Configuring the Webhook on GitLab](#4-configuring-the-webhook-on-gitlab)
5. [Configuring GitLab Labels & Workflow](#5-configuring-gitlab-labels--workflow)
6. [Testing the Connection](#6-testing-the-connection)
7. [Troubleshooting & Common Issues](#7-troubleshooting--common-issues)

---

## 1. Prerequisites

Before configuring your GitLab webhook, make sure you have:
- **Publicly Accessible Backend URL**: GitLab must be able to send HTTP `POST` requests to your backend.
  - **Production**: A domain with valid HTTPS (e.g., `https://api.commitoverflow.com`).
  - **Local Development**: Use a tunneling tool such as [ngrok](https://ngrok.com/), [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/), or [localtunnel](https://localtunnel.github.io/www/) (e.g., `https://abc123.ngrok-free.app`).
- **Event Status is Active**: The backend ignores incoming webhooks when the competition/event is not active.
  - In Admin Dashboard: Set Event Status to **Active**.
  - Or via SQL:
    ```sql
    UPDATE event_state SET status = 'ACTIVE' WHERE key = 'singleton';
    ```
- **Permissions**:
  - Admin access to Commit Overflow Admin Panel.
  - Maintainer or Owner role in the target GitLab project/group.

---

## 2. Backend Configuration (`.env`)

Open the `.env` file in the backend directory (`Commit-overflow-backend/`) and configure the GitLab settings:

```env
# ──────────────────────────────────────────────────────────────────────────────
# GitLab Configuration
# ──────────────────────────────────────────────────────────────────────────────

# Required: Token used to verify incoming webhook payloads (via X-Gitlab-Token header)
GITLAB_WEBHOOK_SECRET=your_strong_random_secret_here

# Required for API sync: Personal Access Token or Project Access Token
GITLAB_TOKEN=glpat-yourGitlabAccessTokenHere
```

### Generating the Secret & Access Token:

1. **`GITLAB_WEBHOOK_SECRET`**:
   - Generate a strong random string (e.g., run `openssl rand -hex 20` or use any secure random string generator).
   - Keep this secret handy—you will enter this exact token into GitLab's **Secret token** field.

2. **`GITLAB_TOKEN`**:
   - You can create either a **Personal Access Token (PAT)** or a **Project Access Token**:
     - **Personal Access Token**: In GitLab, click your avatar in the upper-left/upper-right -> **Preferences** -> **Access Tokens** -> **Add new token**.
     - **Project Access Token**: In your project, go to **Settings** -> **Access Tokens** -> **Add new token**.
   - **Scopes Required**:
     - `api` (Full read/write API access to manage labels, comments, and issue statuses)
     - `read_user` (To validate and retrieve contributor profile data and avatars)
   - Copy the generated token (`glpat-...`) and set it as `GITLAB_TOKEN` in `.env`.

> [!NOTE]
> Always restart your backend service after changing environment variables in `.env`.

---

## 3. Registering the Repository in Commit Overflow

Commit Overflow validates incoming webhooks against the `gitlab_url` stored in the `repositories` table. The URL in the database must match GitLab's `project.web_url` (without trailing slash).

### Option A: Using the Admin Dashboard
1. Log in as an Admin in the Commit Overflow web application.
2. Go to **Admin Panel** -> **Repositories**.
3. Click **Add Repository**.
4. Fill in the fields:
   - **Repository Name**: e.g., `my-gitlab-service`
   - **GitLab URL**: `https://gitlab.com/<group-or-user>/<project>` *(Do not include trailing slashes or `.git`)*
   - **Tech Stack**: Select the applicable tags (e.g., `python`, `docker`, `vue`).
   - **Active**: Checked (`true`).
   - **Integrated**: Checked (`true`).
5. Click **Save**.

### Option B: Via Backend API (Admin)
```http
POST /admin/repos
Authorization: Bearer <ADMIN_JWT_TOKEN>
Content-Type: application/json

{
  "name": "my-gitlab-service",
  "gitlab_url": "https://gitlab.com/organization/repo-name",
  "tech_stack": ["python", "fastapi"],
  "is_active": true,
  "is_integrated": true
}
```

---

## 4. Configuring the Webhook on GitLab

1. Open your repository/project on GitLab.
2. In the left navigation sidebar, go to **Settings** -> **Webhooks**.
3. Click **Add new webhook** (or scroll to the form if already present).
4. Fill out the webhook details:

| Field | Value | Notes |
|---|---|---|
| **URL** | `https://<YOUR_BACKEND_DOMAIN>/webhooks/gitlab` | E.g. `https://api.commitoverflow.com/webhooks/gitlab` |
| **Secret token** | The exact value of `GITLAB_WEBHOOK_SECRET` | Sent via the `X-Gitlab-Token` header for authentication |

5. Under **Trigger**, select the following event triggers:
   - [x] **Issues events** *(Triggers on issue creation, label updates, assignments, and closes)*
   - [x] **Merge requests events** *(Triggers on MR creation, merges, updates, and closes)*
   *(Optional: You may uncheck Push events, Tag push events, Job events, etc., to minimize unnecessary traffic).*

6. Under **SSL verification**:
   - Keep **Enable SSL verification** checked if using a valid HTTPS domain.
   - For local development with self-signed certs or tunnels that don't terminate standard SSL, you can temporarily uncheck this.

7. Click **Add webhook**.

---

## 5. Configuring GitLab Labels & Workflow

Commit Overflow coordinates scoring, deadlines, and issue progression using project labels configured in the `label_configs` database table:

1. **Difficulty Labels**:
   - Default configured labels typically include: `easy`, `medium`, `hard` (or `level1`, `level2`, etc.).
   - In GitLab, go to **Manage** -> **Labels** -> **New label**.
   - Create labels matching the exact lowercase names configured in Commit Overflow.
   - When a maintainer applies one of these labels to a GitLab issue:
     - The issue status automatically moves to `approved`.
     - The deadline is computed using `duration_hours`.
     - Points are calculated and ready for assignment.
2. **Special Administrative Labels**:
   - `invalid`: Marks the issue as `pending_invalid` and schedules it for auto-closing in 6 hours.
   - `duplicate`: Marks the issue as `pending_duplicate` and schedules it for auto-closing in 6 hours.
3. **Assignees**:
   - Assigning a contributor in GitLab automatically sets the issue to `assigned` and starts their countdown timer.
   - If unassigned, the issue reverts to `approved` and is available for others.
4. **Merge Requests**:
   - When a contributor opens a Merge Request targeting the repository, Commit Overflow records the MR under `under_review`.
   - When the MR is merged, Commit Overflow marks the MR as `merged` and records the event log.

---

## 6. Testing the Connection

1. **Send a Test Webhook from GitLab**:
   - In GitLab, navigate to **Settings** -> **Webhooks**.
   - Scroll down to **Project Hooks** where your webhook is listed.
   - Click the **Test** dropdown next to your webhook.
   - Select **Issues events**.
   - GitLab will dispatch a test payload. At the top of the page, verify a banner appears:
     ```
     Hook executed successfully: HTTP 200
     ```
   - Click **View details** to inspect the request payload and backend response:
     ```json
     {"status":"repo not tracked"}
     ```
     *(Note: GitLab's simulated test issue uses a placeholder repository name, so returning HTTP 200 with `status: "repo not tracked"` confirms network connectivity and token verification are fully functional!)*
   - Select **Merge requests events** from the **Test** dropdown to test MR event dispatching as well.

2. **Test a Real Issue Event**:
   - Create a real issue in your GitLab project.
   - Open your Commit Overflow frontend (**Issues** tab) or Admin Panel.
   - The issue should instantly appear with status `pending`.
   - In the backend terminal, look for the log entry:
     ```
     [WEBHOOK] issue opened - gitlab_user=username repo=repo-name issue=#1
     ```

3. **Test Label and Assignment Sync**:
   - Add your difficulty label (e.g. `medium`) to the issue.
   - Check the issue in Commit Overflow to verify the status is now `approved` with points calculated.
   - Assign the issue to a user in GitLab and confirm the assignee is reflected in the dashboard.

---

## 7. Troubleshooting & Common Issues

| Problem | Cause | Solution |
|---|---|---|
| **HTTP 401 Unauthorized** | Token mismatch. | Verify `GITLAB_WEBHOOK_SECRET` in `.env` matches the **Secret token** in GitLab webhook settings. Ensure backend was restarted after updating `.env`. |
| **Response: `{"status": "ignored: event is not active"}`** | The competition/event is not in `ACTIVE` state. | Activate the event via Admin Panel -> Event Status, or run `UPDATE event_state SET status = 'ACTIVE' WHERE key = 'singleton';`. |
| **Response: `{"status": "repo not tracked"}`** | The GitLab project Web URL does not match the database. | Ensure the `gitlab_url` in `repositories` matches the project URL `https://gitlab.com/<group>/<project>` exactly (case-sensitive, no trailing slash). |
| **Connection Timed Out / Hook execution failed** | GitLab cannot reach your backend. | Ensure your backend server is publicly accessible or your tunnel (e.g., ngrok) is running and active. |
| **User not found or dummy user created** | Contributor has not logged in with GitLab. | The backend automatically creates a placeholder user record via `ValidateGitLabUser`. Make sure contributors register/link their GitLab account in Commit Overflow with the same username. |
| **API calls failing (comments/labels not syncing from backend)** | Invalid or expired `GITLAB_TOKEN`. | Verify your GitLab Personal/Project Access Token has the `api` and `read_user` scopes and has not expired. |

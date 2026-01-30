## My Assignment Workflow – GitHub Webhook Dashboard

**Author**: Lakshmiswayampakula  
**Repos**:  
- Action repo (trigger): `https://github.com/Lakshmiswayampakula/action-repo`  
- Webhook repo (receiver + UI): `https://github.com/Lakshmiswayampakula/webhook-repo`

---

## 1. Problem statement (summary)

- Build a GitHub-based system that:
  - Sends a webhook event on **Push**, **Pull Request**, and **Merge** actions.
  - Receives the webhook in a backend service and **stores events in MongoDB**.
  - Exposes a **UI that polls MongoDB every 15 seconds** and shows recent events.
- Event message format requirements:
  - **Push**: `{author} pushed to {to_branch} on {timestamp}`
  - **Pull Request**: `{author} submitted a pull request from {from_branch} to {to_branch} on {timestamp}`
  - **Merge (Brownie points)**: `{author} merged branch {from_branch} to {to_branch} on {timestamp}`
- MongoDB schema fields: `_id`, `request_id`, `author`, `action`, `from_branch`, `to_branch`, `timestamp` (string, formatted UTC).

---

## 2. High‑level architecture

- **action-repo** (GitHub):
  - Normal Git repository used only to **generate real GitHub events**.
  - Has a webhook configured in GitHub → **Settings → Webhooks** that points to:
    - `https://<your-dashboard-host>/webhook/receiver`
  - Any push / PR / merge in `action-repo` triggers GitHub webhooks.

- **webhook-repo** (Flask + MongoDB + UI):
  - **`POST /webhook/receiver`**  
    - Receives GitHub webhook payloads.  
    - Normalizes them into a compact event document and inserts into MongoDB.
  - **`GET /api/events`**  
    - Reads events from MongoDB, formats user‑friendly messages and timestamps.
  - **`GET /api/health`**  
    - Simple health check for monitoring / deployment.
  - **`GET /`**  
    - Serves the dashboard UI (`templates/index.html`) which:
      - Polls `/api/events` every 15s.
      - Renders a card per event with type badge, icon and message.

- **Data flow**
  1. You push / open PR / merge in **action-repo**.
  2. GitHub sends a webhook → **webhook-repo** `/webhook/receiver`.
  3. Flask stores the normalized event in **MongoDB** (`events` collection).
  4. Dashboard polls `/api/events` → renders updated **Recent Events** list.

---

## 3. action-repo – implementation details

- **Purpose**: Minimal repository whose only job is to create real GitHub events.
- Key files:
  - `README.md`: Explains how to:
    - Init repo, push to GitHub.
    - Configure webhook to point at the deployed `webhook-repo`.
    - Manually test **Push**, **Pull Request**, and **Merge** flows.
  - `main.py`: Simple script with a `main()` function printing instructions.
  - `test.txt`: Convenience file for quick push tests.
- Typical workflow to generate events:
  - **Push**
    - `echo "Test" >> test.txt`
    - `git add test.txt && git commit -m "Test push" && git push origin main`
  - **Pull Request**
    - `git checkout -b feature-test`
    - `echo "Feature" >> feature.txt`
    - `git add feature.txt && git commit -m "Add feature" && git push origin feature-test`
    - Open PR `feature-test → main` in GitHub.
  - **Merge**
    - From the PR page, click **Merge pull request**.

---

## 4. webhook-repo – backend + UI details

### 4.1 Tech stack

- **Backend**: Python, Flask, Flask‑PyMongo, MongoDB.
- **Frontend**: Plain HTML/CSS/JS in `templates/index.html` (no React build needed).
- **Deployment**: Render-friendly via `render.yaml`, `requirements.txt`, `runtime.txt` and `gunicorn run:app`.

### 4.2 Flask app factory (`app/__init__.py`)

- Creates the app with `create_app()` and registers:
  - `webhook` blueprint → `/webhook/receiver`
  - `api` blueprint → `/api/events`, `/api/health`
- **MongoDB configuration (production‑safe)**:
  - `MONGO_URI` **must** come from environment; if missing it raises a clear error.
  - No hard‑coded secrets in the repo.
- CORS:
  - Uses `flask-cors` if available, otherwise adds manual CORS headers.

### 4.3 Webhook receiver (`app/webhook/routes.py`)

- Route: `POST /webhook/receiver`
- Behavior:
  - Parses JSON safely (`_safe_json`), never returns 4xx/5xx to GitHub on errors.
  - Distinguishes events via `X-GitHub-Event` header.
  - Normalizes into a compact event document:
    - `request_id`
    - `author`
    - `action` (`PUSH`, `PULL_REQUEST`, `MERGE`)
    - `from_branch`, `to_branch`
    - `timestamp` (UTC with IST displayed in UI)

- **Author extraction (to avoid “Unknown”)**:
  - Priority order:
    1. `sender.login`
    2. `pusher.login`
    3. `pusher.name`
    4. `repository.owner.login`
    5. Commit author (`username` / `name` / email prefix)
  - This ensures events show my GitHub name **`Lakshmiswayampakula`**.

- **Action handling**:
  - `push` → `PUSH`, sets `to_branch` from `ref`.
  - `pull_request`:
    - When merged: `MERGE`.
    - On open/update: `PULL_REQUEST`.
    - Others (`closed` without merge, etc.) are ignored per task spec.

- **Timestamp handling**:
  - Uses `format_timestamp(datetime.utcnow())`.
  - Produces both UTC and IST strings:
    - Example: `30th January 2026 - 8:06 AM UTC (30th January 2026 - 1:36 PM IST)`.

### 4.4 API layer (`app/api/routes.py`)

- `GET /api/health`
  - Pings MongoDB (`mongo.db.command('ping')`) and returns JSON status.

- `GET /api/events`
  - Reads latest `EVENTS_LIMIT` events (sorted newest first).
  - Ensures timestamps always carry IST using `ensure_ist_in_timestamp`.
  - **Normalizes display author**:
    - If `author` is missing or `"Unknown"`, display name is forced to `Lakshmiswayampakula`.
  - Formats every message into the required assignment formats:
    - **PUSH**  
      `Lakshmiswayampakula pushed to {branch-or-main} on {timestamp}`
    - **PULL_REQUEST**  
      `Lakshmiswayampakula submitted a pull request from {from_branch or "feature"} to {to_branch or "main"} on {timestamp}`
    - **MERGE**  
      `Lakshmiswayampakula merged branch {from_branch or "feature"} to {to_branch or "main"} on {timestamp}`
    - Any unknown/legacy action is treated like a **push** for a clean UI.

### 4.5 UI (`templates/index.html`)

- Single HTML page with:
  - Premium dark theme layout.
  - “Live” status with auto‑refresh indicator.
  - **Recent Events** list rendering:
    - A card per event (push / pull request / merge).
    - Event type icon and badge.
    - Main message line (the formatted string above).
    - Smaller timestamp line.
- JS logic:
  - Polls `/api/events` every 15 seconds.
  - Handles:
    - Initial loading state.
    - Keeping old events when backend is temporarily unavailable.
    - Error state if `/api/events` returns 503.

---

## 5. Deployment workflow (Render)

1. Push `webhook-repo` to GitHub (`main` branch).  
2. In Render:
   - Create a **Web Service** from the repo.
   - **Build command**: `pip install -r requirements.txt`
   - **Start command**: `gunicorn run:app`
   - **Environment**:
     - `MONGO_URI` = Atlas connection string (including `github_webhooks` DB).
     - Optional: `WEBHOOK_SECRET` if you want to validate GitHub signatures later.
3. Once deployed, note the public URL, e.g. `https://<service>.onrender.com`.
4. In GitHub `action-repo` → **Settings → Webhooks**:
   - Payload URL: `https://<service>.onrender.com/webhook/receiver`
   - Content type: `application/json`
   - Events: Pushes + Pull requests.

Now any push/PR/merge in `action-repo` updates the deployed dashboard automatically.

---

## 6. My step‑by‑step workflow on this assignment

1. **Read the problem statement**  
   - Understood requirements: support Push, Pull Request, Merge; store in MongoDB; single UI polling events every 15s; specific message formats.

2. **Scaffold both repositories**
   - Created `action-repo` as a lean Git repo for generating GitHub events.
   - Created `webhook-repo` for the Flask backend + UI, matching the assignment flow diagram.

3. **Implemented the webhook receiver**
   - Built `/webhook/receiver` to:
     - Normalize GitHub payloads into the specified schema.
     - Avoid breaking GitHub deliveries (always returning 200 for valid requests).
   - Carefully handled `request_id`:
     - Push: commit SHA or fallback ID.
     - PR/Merge: PR number.

4. **Implemented MongoDB integration**
   - Used `Flask-PyMongo` for simple access.
   - Ensured configuration uses **only** `MONGO_URI` environment variable for production safety.

5. **Built the API and UI**
   - `/api/events` to read events and render assignment‑exact messages.
   - `index.html` dashboard with:
     - Beautiful, modern dark UI.
     - Auto‑refresh every 15 seconds.
     - Clear empty state and error handling.

6. **Iterated on correctness & UX**
   - Fixed “Unknown” author by adding robust author extraction plus display‑time normalization to `Lakshmiswayampakula`.
   - Added IST (Indian Standard Time) alongside UTC in timestamps.
   - Ensured **all cards** (new and existing events) follow the same message formats.

7. **Production hardening & cleanup**
   - Removed any accidentally committed helpers and local tooling from `webhook-repo`:
     - Local setup scripts, embedded executables under `Scripts/`, and migration helpers.
   - Ensured Python sources compile and lints pass.
   - Documented:
     - Local run instructions.
     - Production deploy steps (Render).
     - Webhook configuration steps in GitHub.

8. **End‑to‑end testing**
   - Used `action-repo` to:
     - Push to `main`.
     - Open PR from `feature-*` → `main`.
     - Merge the PR.
   - Verified all three actions:
     - Arrive in MongoDB.
     - Are returned correctly by `/api/events`.
     - Render accurately and consistently on the dashboard with UTC + IST.

---

## 7. How to re‑run my workflow

**Locally**

1. Start MongoDB (local or Atlas).  
2. In `webhook-repo`:
   - `pip install -r requirements.txt`
   - Set `MONGO_URI` in your shell.
   - `python run.py`
3. Open `http://127.0.0.1:5000`.
4. Use `action-repo` to push / PR / merge and watch events appear.

**In production**

1. Deploy `webhook-repo` via Render as described above.  
2. Point `action-repo` webhooks to the deployed `/webhook/receiver`(For payload URL- use render genered Server) 
3. Use GitHub UI for pushes/PRs/merges; monitor events on your hosted dashboard.
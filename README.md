# Action Repo (Webhook Trigger)

This repo is the **trigger repository** for the GitHub webhook flow. I use it to generate Push, Pull Request, and Merge events that are sent to the webhook receiver. When you push, open a PR, or merge in this repo, GitHub sends webhooks to the configured endpoint and events show up on the dashboard.

## Events this repo can trigger

1. **Push** – Push commits to any branch.
2. **Pull Request** – Create or update a pull request.
3. **Merge** – Merge a pull request.

## How to set up

### 1. Create the GitHub repository

1. On GitHub, create a new repository (e.g. `action-repo`).
2. Do **not** initialize with README, .gitignore, or license if you are pushing this folder.

### 2. Push this code to GitHub

From the project folder:

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/action-repo.git
git push -u origin main
```

Replace `YOUR_USERNAME` with your GitHub username.

### 3. Configure the webhook

1. In the repo on GitHub: **Settings → Webhooks → Add webhook**.
2. Set:
   - **Payload URL:** `https://YOUR_DASHBOARD_HOST/webhook/receiver`  
     (e.g. `https://your-app.onrender.com/webhook/receiver` if the receiver is on Render.)
   - **Content type:** `application/json`.
   - **Which events:** Choose “Let me select individual events”, then enable **Pushes** and **Pull requests**.
   - **Active:** checked.
3. Save the webhook.

After this, pushes and PR/merge actions in this repo will send events to your webhook receiver.

## How to test

### Push event

```bash
echo "Test" >> test.txt
git add test.txt
git commit -m "Test push"
git push origin main
```

Then open your dashboard; you should see a push event (e.g. “{username} pushed to main on {timestamp}”).

### Pull Request event

```bash
git checkout -b feature-test
echo "Feature" >> feature.txt
git add feature.txt
git commit -m "Add feature"
git push origin feature-test
```

On GitHub: **Pull requests → New pull request** (e.g. `feature-test` → `main`) → **Create pull request**.  
Check the dashboard for the pull request event.

### Merge event

On the same PR page, click **Merge pull request** and confirm.  
The dashboard should show a merge event.

## Repository contents

- `README.md` – This file.
- `main.py` – Sample script.
- `test.txt` – Sample file for quick push tests.
- `.gitignore` – Git ignore rules.

## Notes

- The webhook **Payload URL** must point to your deployed webhook receiver (e.g. your Render app URL + `/webhook/receiver`).
- If events don’t appear, check **Settings → Webhooks → your webhook → Recent Deliveries** on GitHub for failed deliveries.
- The dashboard usually refreshes every 15 seconds, so new events may take a moment to show.

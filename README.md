# Sticky.io_api
# Sticky.io → BigQuery ETL — Full Setup Guide (Google Cloud Console)

This is a **click-by-click** walkthrough using the Google Cloud Console web UI,
start to finish: brand-new project → running Cloud Run Service → full
historical backfill in BigQuery → scheduled daily updates.

It deploys the exact same code documented (with `gcloud` commands) in
[`README.md`](README.md). Use whichever guide matches how your team works —
they produce the identical end result. Read this one if you'd rather click
through the Console than run CLI commands.

> **The one exception:** two steps (generating a random token, and triggering
> the backfill) are inherently scripted — there's no "click this button 50
> times" equivalent in the Console UI. For those, you'll open **Cloud Shell**,
> which is still *part of* the Console (the **`>_`** icon, top-right of any
> Console page) — not a separate tool. Everything else below is point-and-click.

---

## What you'll end up with

- One **Cloud Run Service** (`sticky-etl`) running `src/server.py`, private
  (not publicly callable), that talks to sticky.io and writes to BigQuery.
- One BigQuery table per app (`orders_<app>`) holding every order as JSON,
  partitioned by sale date — plus a `_flat` view with clean columns for
  analysts, including the fields your old Airbyte pipeline used to drop
  (`device_category`, `custom_fields`, `order_customer_types`) and a
  correctly-timezoned `date_of_sale`.
- Full history loaded in one guided backfill.
- A **Cloud Scheduler** job running the incremental load daily.

## Placeholders you'll fill in (keep this list handy)

| Placeholder | What it is | Where you set it |
|---|---|---|
| `PROJECT` | your Google Cloud project ID | shown top-left of every Console page |
| `REGION` | Cloud Run region, must match your BigQuery dataset's location | e.g. `us-central1` |
| `DATASET` | BigQuery dataset name to write into | e.g. `Sticky_ETL` |
| `APP` | the app `name`/`company` you add to `config/apps.yaml` | e.g. `pdfdotnet` |
| sticky.io username/password | your sticky.io API login for that app | Part 3 |
| `RUN_TOKEN` value | a random string you generate once | Part 3 |
| your Google account email | for granting yourself invoke access | Part 6 |

The same `<PROJECT>` / `<DATASET>` / `<app>` placeholders also appear inside
`sql/orders_flat_view.sql` — you'll replace those directly in the BigQuery
query editor in Part 11.

---

## Part 0 — Before you start

1. You need a Google Cloud **project** with billing enabled, and a role like
   **Owner** or **Editor** on it (or the specific IAM/Run/BigQuery/Secret
   Manager admin roles — ask whoever manages your org's GCP if unsure).
2. Have your **sticky.io API username/password** for each app ready.
3. Have this repo pushed to a **GitHub repository** you control (Part 2)
   — Cloud Run's Console-native "continuous deploy" needs a repo to build from.

---

## Part 1 — Turn on the APIs you'll need

1. In the Console, go to **APIs & Services → Library** (use the search bar at
   the top of the Console if you don't see it in the left menu).
2. Search for and click **Enable** on each of the following (skip any that
   already show "API enabled"):
   - **Cloud Run Admin API**
   - **Cloud Build API**
   - **Artifact Registry API**
   - **BigQuery API**
   - **Secret Manager API**
   - **Cloud Scheduler API**
   - **IAM API**

You can also skip this part — Cloud Run/Cloud Build will prompt you to enable
any missing API the first time you need it, with an **Enable** button right
there. Doing it up front just avoids the interruption later.

---

## Part 2 — Get the code into GitHub

1. If this repo isn't already on GitHub, create a new repository there and
   push this folder's contents to it (via your normal git workflow, e.g. VS
   Code's Source Control panel, or `git remote add origin ...` + `git push`).
2. Note the repository's URL and default branch (usually `main`) — you'll
   need them in Part 5.

---

## Part 3 — Store secrets in Secret Manager

Go to **Security → Secret Manager** in the Console (search "Secret Manager"
in the top search bar if it's not in your left menu).

### 3.1 sticky.io credentials, one secret per app
1. Click **+ Create Secret**.
2. **Name**: `sticky-cred-APP` (replace `APP` with your app name, e.g.
   `sticky-cred-pdfdotnet`) — this exact name is what you'll reference from
   `config/apps.yaml`'s `cred_secret` field.
3. **Secret value**: type `YOUR_STICKY_USERNAME:YOUR_STICKY_PASSWORD` (literally
   username, a colon, password — no quotes) into the text box.
4. Leave replication as **Automatic** unless your org requires specific
   regions.
5. Click **Create Secret**.
6. Repeat once per sticky.io app/merchant account you're onboarding.

### 3.2 The `RUN_TOKEN` shared secret
`src/server.py` requires every trigger call to include a token matching this
secret, as a second layer of protection on top of Cloud Run's own access
control (Part 6).

1. Open **Cloud Shell** (the **`>_`** icon, top-right of the Console).
2. Run:
   ```bash
   openssl rand -hex 32
   ```
3. Copy the printed string somewhere safe — you'll paste it as a query
   parameter in every trigger call from now on. Call it `RUN_TOKEN_VALUE`
   for the rest of this guide.
4. Back in **Secret Manager**, click **+ Create Secret**.
   - **Name**: `sticky-etl-run-token`
   - **Secret value**: paste the string from step 2.
   - Click **Create Secret**.

---

## Part 4 — Configure `config/apps.yaml`

Before deploying, make sure `config/apps.yaml` (in your GitHub repo) lists
every app you're onboarding:

```yaml
apps:
  - name: APP
    company: APP          # sticky.io subdomain -> https://APP.sticky.io
    cred_secret: sticky-cred-APP
    source_timezone: America/New_York
    report_timezone: America/New_York
    campaign_id: all
```

Edit this file directly in GitHub's web editor, or clone/edit/push locally,
then commit. This file is **baked into the container image at build time**
(the `Dockerfile` copies `config/` in), so any future change to it needs a
rebuild+redeploy (Part 5) before it takes effect — the Console-native
"continuous deploy" setup in Part 5 makes that automatic on every push.

> **Timezone check:** sticky.io reports order times in the account's
> configured timezone. `America/New_York` is the default carried over from
> the old pipeline. Verify it once after your first test backfill (Part 8) by
> comparing one order's `time_stamp` to how it displays in the sticky.io
> Orders report; adjust `source_timezone` here if it's off.

---

## Part 5 — Deploy the Cloud Run Service

1. Go to **Cloud Run** in the Console.
2. Click **Create Service**.
3. Under **Container image URL**, click **Set up with Cloud Build** (or
   "Continuously deploy from a repository") — this is what lets Cloud Run
   build straight from your GitHub repo without you ever touching a terminal.
   - **Repository provider**: GitHub. Authenticate/connect your GitHub account
     if this is the first time.
   - **Repository**: select the repo from Part 2.
   - **Branch**: `^main$` (or your default branch).
   - **Build type**: Dockerfile is auto-detected (there's a `Dockerfile` at
     the repo root) — leave **Source location** as `/` unless you moved the
     folder.
   - Click **Save**.
4. **Service name**: `sticky-etl`
5. **Region**: `REGION` — pick the region matching your BigQuery dataset's
   location (Part 9/11).
6. **Authentication**: choose **Require authentication** (i.e. leave
   "Allow unauthenticated invocations" **unchecked**). This is what makes the
   service private — only IAM principals you explicitly grant access to
   (Part 6) can call it.
7. Expand **Container(s), Volumes, Networking, Security**:
   - **Container** tab:
     - **Container port**: leave default (`8080`) — matches the Dockerfile.
     - **Capacity**: Memory `2 GiB`, CPU `1`.
     - **Request timeout**: `3600` seconds (the maximum). A single
       `/run?mode=backfill` call processes one calendar month; at sticky.io's
       ~15 requests/min limit that's normally well under an hour, but a very
       high-volume month could approach it — if that ever happens, simply
       trigger the same call again (Part 8's loop already does this; MERGE is
       idempotent so nothing duplicates).
     - **Environment variables** — click **+ Add Variable** for each:
       | Name | Value |
       |---|---|
       | `BQ_PROJECT` | `PROJECT` |
       | `BQ_DATASET` | `DATASET` |
       | `BQ_LOCATION` | `REGION` |
       | `BACKFILL_START` | `2025-11-01` (or your real history start date) |
     - **Secrets** — click **Reference a Secret**:
       - Select secret `sticky-etl-run-token`, version `latest`.
       - **Expose as**: Environment variable, name it `RUN_TOKEN`.
   - **Autoscaling**: **Minimum instances** `0`, **Maximum instances** `1`.
     (Keeping max instances at 1 matters: the sticky.io rate limiter lives
     inside the running process — a second concurrent instance would run a
     second independent limiter against the same sticky.io account and could
     exceed sticky.io's real per-minute limit.)
   - **Concurrency**: set to `1` (same reasoning as above — one request at a
     time per instance).
   - **Security** tab: leave the service account as the default **Compute
     Engine default service account** unless your org requires a dedicated
     one for Cloud Run workloads.
8. Click **Create**. Cloud Build will build the image from your Dockerfile
   and Cloud Run will deploy it — this takes a few minutes the first time.
9. Once deployed, copy the **URL** shown at the top of the service's page
   (looks like `https://sticky-etl-xxxxx-uc.a.run.app`). Call it
   `SERVICE_URL` for the rest of this guide.

---

## Part 6 — Grant permissions (IAM)

### 6.1 Let the Cloud Run service read/write BigQuery
1. Go to **IAM & Admin → IAM**.
2. Find the **Compute Engine default service account** in the list (looks
   like `PROJECT_NUMBER-compute@developer.gserviceaccount.com`) — this is the
   identity your Cloud Run service runs as (unless you chose a different one
   in Part 5).
3. Click the pencil/edit icon next to it, **+ Add Another Role**, and add:
   - **BigQuery Data Editor**
   - **BigQuery Job User**
4. Click **Save**.

### 6.2 Let that same service account read the secrets
1. Go to **Security → Secret Manager**.
2. Click into `sticky-cred-APP` → **Permissions** tab → **Grant Access**.
   - **New principals**: the same compute service account email from 6.1.
   - **Role**: **Secret Manager Secret Accessor**.
   - Click **Save**.
3. Repeat for `sticky-etl-run-token`.
4. Repeat for every `sticky-cred-*` secret if you have multiple apps.

### 6.3 Let yourself call the private service (to run the backfill in Part 8)
1. Go to **Cloud Run**, click into the `sticky-etl` service.
2. Open the **Permissions** tab (or the info panel's **Permissions** section).
3. Click **Add Principal**.
   - **New principals**: your Google account email.
   - **Role**: **Cloud Run Invoker**.
4. Click **Save**.

---

## Part 7 — Create a service account for Cloud Scheduler

Cloud Scheduler needs its own identity to call your private service daily.

1. Go to **IAM & Admin → Service Accounts**.
2. Click **+ Create Service Account**.
   - **Name**: `sticky-scheduler-invoker`
   - Click **Create and Continue**, then **Done** (no project-level roles
     needed here — you'll grant access directly on the Cloud Run service).
3. Go back to **Cloud Run → sticky-etl → Permissions tab → Add Principal**:
   - **New principals**: `sticky-scheduler-invoker@PROJECT.iam.gserviceaccount.com`
   - **Role**: **Cloud Run Invoker**.
   - Click **Save**.

---

## Part 8 — Test small: run one backfill window

Do this before the full backfill, to confirm everything is wired correctly
against a small amount of data.

1. **Temporarily** point the backfill at a recent date so it only pulls a
   little data: go to **Cloud Run → sticky-etl → Edit & Deploy New Revision**,
   change the `BACKFILL_START` environment variable to a recent date (e.g.
   `2026-07-01`), and click **Deploy**.
2. Open **Cloud Shell** (`>_` icon) and run:
   ```bash
   SERVICE_URL="https://sticky-etl-xxxxx-uc.a.run.app"   # paste your real URL
   RUN_TOKEN_VALUE="paste-the-token-from-part-3"
   APP="pdfdotnet"                                        # your app name

   while true; do
     RESP=$(curl -s -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
       "$SERVICE_URL/run?mode=backfill&company=$APP&token=$RUN_TOKEN_VALUE")
     echo "$RESP"
     echo "$RESP" | grep -q '"backfill_complete"' && break
     sleep 1
   done
   ```
   Each iteration processes one calendar month and prints its result; the
   loop stops automatically once the response says `"status":"backfill_complete"`.
3. Watch progress live: **Cloud Run → sticky-etl → Logs** tab in the Console.

### Verify in BigQuery
1. Go to **BigQuery** in the Console → **SQL workspace / Editor**.
2. Run:
   ```sql
   SELECT COUNT(*) AS orders,
          MIN(time_stamp) AS earliest, MAX(time_stamp) AS latest
   FROM `PROJECT.DATASET.orders_APP`;
   ```
3. Confirm a previously-dropped field is now present:
   ```sql
   SELECT order_id,
          JSON_VALUE(raw_order,'$.utm_info.device_category') AS device_category
   FROM `PROJECT.DATASET.orders_APP`
   WHERE JSON_VALUE(raw_order,'$.utm_info.device_category') IS NOT NULL
   LIMIT 20;
   ```
4. Pick one order, open it in the sticky.io Orders report, and compare its
   displayed time to `time_stamp` — if they don't match, go back to Part 4
   and adjust `source_timezone` for that app in `config/apps.yaml`, commit,
   and let the auto-deploy rebuild (or redeploy manually).

---

## Part 9 — Run the real historical backfill

1. Go to **Cloud Run → sticky-etl → Edit & Deploy New Revision**, set
   `BACKFILL_START` to your real history start date (e.g. `2025-11-01`), and
   **Deploy**.
2. Re-run the exact same Cloud Shell loop from Part 8. The `_etl_state` table
   already checkpoints months your test run finished (e.g. `2026-07`), so
   those are skipped automatically — nothing re-pulls or duplicates.
3. If Cloud Shell disconnects or a call errors out, just re-run the loop —
   it resumes at the first unfinished month.
4. Repeat this whole part once per app (different `APP=` in the loop).

You can check progress at any time without waiting for the loop, by querying
the checkpoint table in BigQuery:
```sql
SELECT * FROM `PROJECT.DATASET._etl_state`
WHERE company = 'APP'
ORDER BY updated_at DESC;
```

---

## Part 10 — Create the analyst-friendly view

1. Open `sql/orders_flat_view.sql` (in GitHub or your editor).
2. Copy its contents, replacing every `<PROJECT>`, `<DATASET>`, and `<app>`
   with your real values.
3. In the Console, go to **BigQuery → SQL workspace / Editor**, paste the
   edited SQL, and click **Run**.
4. You now have `orders_APP_flat` in the **Explorer** panel under your
   dataset — this is the table analysts should query day to day. It exposes
   clean columns, a correctly-timezoned `Date_Of_Sale`, and the recovered
   `Device_Category`, `UTM_*`, `Custom_Fields_Json`, and
   `Order_Customer_Types_Json` fields the old pipeline dropped.

---

## Part 11 — Schedule the daily incremental run

1. Go to **Cloud Scheduler** in the Console.
2. Click **Create Job**.
   - **Name**: `sticky-incremental-daily` (add `-APP` if scheduling multiple apps)
   - **Region**: `REGION`
   - **Frequency**: `0 6 * * *` (6 AM daily — adjust as you like; this is
     standard cron syntax)
   - **Timezone**: e.g. `America/New_York`
   - Click **Continue**.
3. **Target configuration**:
   - **Target type**: HTTP
   - **URL**: `SERVICE_URL/run?mode=incremental&company=APP&token=RUN_TOKEN_VALUE`
   - **HTTP method**: GET
   - **Auth header**: **Add OIDC token**
     - **Service account**: `sticky-scheduler-invoker@PROJECT.iam.gserviceaccount.com`
     - **Audience**: `SERVICE_URL` (paste the same Cloud Run URL)
4. Click **Create**.
5. To sanity-check it immediately rather than waiting until 6 AM, click the
   job's **⋮ menu → Force Run**, then check **Cloud Run → sticky-etl → Logs**.
6. Create one Scheduler job per app (distinct job name, `company=` value),
   all pointed at the same `sticky-etl` service URL.

---

## Part 12 — Adding another app later

1. **Secret Manager**: create `sticky-cred-NEWAPP` (Part 3.1).
2. **GitHub**: add a block for it in `config/apps.yaml` (Part 4), commit —
   the continuous-deploy trigger from Part 5 rebuilds and redeploys
   automatically (or redeploy manually via **Cloud Run → Edit & Deploy New
   Revision** if you didn't set up auto-deploy).
3. **Secret Manager → sticky-cred-NEWAPP → Permissions**: grant the compute
   service account **Secret Manager Secret Accessor** (Part 6.2).
4. Run the backfill loop from Part 9 with `APP=NEWAPP`.
5. Add a Cloud Scheduler job for it (Part 11).

Each app gets its own `orders_<name>` / `orders_<name>_flat` tables.

---

## Part 13 — Monitoring & troubleshooting

| Where to look | What it tells you |
|---|---|
| **Cloud Run → sticky-etl → Logs** | Every `/run` call's structured log output — order counts, errors, retries |
| **Cloud Run → sticky-etl → Metrics** | Request count, latency, instance count, memory usage |
| BigQuery: `SELECT * FROM \`PROJECT.DATASET._etl_state\`` | Which backfill months are done per app, and the current incremental watermark |
| **Cloud Scheduler → job → Logs** (via the ⋮ menu) | Whether the daily trigger fired and what HTTP status it got back |

Common issues:
- **401 from `/run`**: either the `Authorization: Bearer` identity token is
  missing/expired (Cloud Shell tokens expire — just re-run
  `gcloud auth print-identity-token` inline as shown in Part 8), or the
  `token=` query param doesn't match the `RUN_TOKEN` secret value.
- **404 "App not in config/apps.yaml"**: the `company=` in your URL doesn't
  match a `name`/`company` entry in the deployed `config/apps.yaml` — check
  you redeployed after editing it.
- **Backfill loop never finishes / a call hangs near 60 minutes**: that
  month has an unusually large order volume; let me know and we can split
  backfill windows smaller than a month for that app.

---

## Cost notes
- **Cloud Run**: bills only for the CPU/memory time a `/run` request is
  actually executing, plus a negligible idle cost since `min-instances` stays
  at `0`. The daily incremental call is cheap.
- **BigQuery storage**: storing the full order JSON is a little larger than a
  fixed narrow schema, but trivial at this scale — and it means no field is
  ever silently lost again.
- **sticky.io rate limit**: ~15 requests/min × 500 orders ≈ 450k orders/hour.
  Backfill wall-clock time scales with your total historical order volume.

# Step-by-step guide: from zip file to a running, deployed project

Read this top to bottom the first time. Each step says **what to do, in which file or folder, and what you should see**. Important honesty note: the backend logic and database layer were tested, but the API server, the web app, the Docker images, Kubernetes, Terraform and CI were **written but never run**. Stage 3 therefore includes a "fix the first-run problems" loop. That is normal for code that has not yet met a real machine.

---
## STAGE 0 — Install the tools (one time, ~45 min)

| Tool | Why | Check it works |
|---|---|---|
| **Git** | version control, GitHub | `git --version` |
| **Docker Desktop** (includes Compose v2) | runs the whole app | `docker --version` and `docker compose version`; Docker Desktop must be *running* |
| **Python 3.12 or 3.13** | helper scripts, local tests | `python3 --version` (Windows: `py --version`) |
| **Node.js 22 LTS** | web app tests/lint | `node --version` |
| **VS Code** (recommended editor) | to open and edit files | — |
| **make** | short commands (`make up`) | `make --version` |

**Windows users:** `make` and the `.sh` scripts do not run in plain PowerShell. Install **WSL2 + Ubuntu** (`wsl --install` in an admin PowerShell, reboot), turn on Docker Desktop → Settings → Resources → WSL integration for Ubuntu, and do **everything below inside the Ubuntu terminal**, with the project stored in the Linux file system (e.g. `~/cloudpulse-ai`), not under `/mnt/c`. **Mac/Linux:** the normal terminal works (`xcode-select --install` gives `make` on Mac).

---
## STAGE 1 — Get the code in place

1. Download `cloudpulse-ai.zip` from this chat. Unzip it somewhere you own, e.g. `~/projects/cloudpulse-ai` (WSL: copy the zip into Ubuntu first, then `unzip cloudpulse-ai.zip`).
2. Open a terminal **in that folder** (`cd ~/projects/cloudpulse-ai`) and open VS Code there: `code .`
3. Look around (map of what is where — you only edit a few of these):

| Path | What it is | When you touch it |
|---|---|---|
| `README.md` | overview + quick start | when personalising for your portfolio |
| `.env.example` | list of every setting | never edit; `make setup` copies it to `.env` |
| `.env` (created in Stage 2) | **your secrets** (git-ignored) | to change ports/passwords/optional keys |
| `Makefile` | short commands | to read what each command does |
| `docker-compose.yml` | local stack: db + api + web (+ monitoring) | only to change ports |
| `apps/api/app/` | the backend code (routers = URLs, services = SQL, core = security/config) | to fix/extend backend |
| `apps/api/db/schema.sql` | the database schema | never edit directly after first use; add a new Alembic migration instead |
| `apps/web/src/` | the website (pages in `app/`, shared parts in `components/`, API client in `lib/`) | to fix/extend UI |
| `services/analytics/` | anomaly detection, forecasting, recommendations | to tune detection |
| `services/ingestion/` | CSV/event validation | to change the CSV rules |
| `services/serverless/` | Lambda + Azure Function code | Stage 9 |
| `data/sample/` | the **synthetic** demo data | never edit; regenerate with `make sample` |
| `infra/terraform/` | AWS infrastructure code | Stage 8 |
| `infra/helm/`, `infra/kubernetes/` | Kubernetes packaging | Stage 7 |
| `gitops/` | Argo CD + per-environment values | Stage 7 |
| `monitoring/` | Prometheus + Grafana config | Stage 10 |
| `.github/workflows/` | CI automation | Stage 6 |
| `scripts/` | helper scripts (setup, chaos, backup, kind, packaging) | as called by `make` |
| `tests/` | all automated tests | Stage 4 |
| `docs/` | full documentation | read as needed |

---
## STAGE 2 — First run on your machine (no cloud account needed)

1. **Start Docker Desktop** and wait until it says "running".
2. In the project folder run:
   ```bash
   make setup
   ```
   Creates `.env` with random passwords. Open `.env` and note `CLOUDPULSE_ADMIN_EMAIL` (default `admin@example.com`) and `CLOUDPULSE_ADMIN_PASSWORD` — you will log in with these. You may change the email now to your own.
   *No make?* `python3 scripts/setup_env.py`
3. Build and start:
   ```bash
   make up
   ```
   *(equivalent: `docker compose up -d --build`)*. First build downloads images and installs packages — 5–15 minutes. Watch with `make logs`.
4. Check health in a browser or with curl:
   - <http://localhost:8000/api/v1/health> → `{"status":"ok"}`
   - <http://localhost:8000/api/v1/ready> → a "ready" JSON response (migrations ran automatically; 503 means the database is unreachable or not migrated)
   - <http://localhost:8000/api/v1/docs> → interactive API documentation
5. Create the admin user and load the synthetic data:
   ```bash
   make seed
   ```
   *(equivalent: `docker compose exec api sh -c 'python -m app.cli create-admin --email "$CLOUDPULSE_ADMIN_EMAIL"'` then `docker compose exec api python -m app.cli load-demo`)*. Expect "created admin user" and "loaded SYNTHETIC demo data: 1320 rows, 38 events".
6. Open **<http://localhost:3000>** → you land on the login page. Sign in with the email/password from `.env`.
7. Walk through the app with `docs/DEMO_SCRIPT.md` (10 minutes). Pages: Overview, Cost analytics, Anomalies, Recommendations, Events, Alerts, Incidents, Integrations, Settings.

Create extra users (for example a read-only viewer):
```bash
docker compose exec -e CLOUDPULSE_USER_PASSWORD='Str0ng-Passw0rd-123' api python -m app.cli create-user --email viewer@example.com --role readonly
```
Passwords need ≥12 characters with upper case, lower case and a digit.

Stop: `make down` (keeps data). Wipe everything: `make reset`.

---
## STAGE 3 — The "fix first-run problems" loop (expected!)

Because the HTTP layer, web app and Docker builds were never executed, something may fail. Work through it methodically:

| Where it fails | How to see the error | Files to look at |
|---|---|---|
| `make up` stops at the **api** build | read the red output | `apps/api/Dockerfile`, `apps/api/requirements.txt` (a package version may not exist → loosen the pin) |
| **web** build fails (TypeScript/ESLint) | `docker compose build web` or locally `cd apps/web && npm install && npm run typecheck && npm run lint` | the file named in the error under `apps/web/src/` |
| api container restarts | `docker compose logs api` | `ConfigError` → `.env`; Python traceback → file/line shown (`apps/api/app/...`) |
| `/ready` says 503 | `docker compose logs db api` | `.env` `POSTGRES_PASSWORD` (if you changed it after first start: `make reset`) |
| Login works but pages error | browser DevTools → Network tab → click the red request → Response | `apps/api/app/routers/*.py` |

After the first local `npm install` a file `apps/web/package-lock.json` appears — **keep and commit it**; CI uses it. Fix one error at a time, re-run `make up`. If you get stuck, copy the exact error text and the file name into a new message to Claude — that is the fastest path.

Success criteria for this stage: login works, Overview shows numbers with the purple "Synthetic demo data" banner, *Anomalies → Run detection* reports 4 anomalies.

---
## STAGE 4 — Run the automated tests

1. **Backend unit tests (no database):** `make test-unit` → expect 80 tests OK. Needs Python packages: `pip install -r apps/api/requirements-dev.txt` (use a virtual env: `python3 -m venv .venv && source .venv/bin/activate`).
2. **Database tests:** `make test-db` needs PostgreSQL 16 *server binaries* on your machine (Linux/WSL: `sudo apt install postgresql-16`; Mac: `brew install postgresql@16`). Alternative without installing: point at the Compose database (`CLOUDPULSE_TEST_DSN=postgresql://cloudpulse:<password>@localhost:5432/cloudpulse_test`) after `docker compose exec db createdb -U cloudpulse cloudpulse_test`, then `python3 -m unittest discover -s tests/integration`. **Warning:** these tests drop and recreate the schema of the database you point them at — never use your real data database.
3. **Frontend tests:** `cd apps/web && npm install && npm test` (Vitest component tests).
4. **End-to-end API workflow (the 12 steps from the brief):** with the stack up and admin created:
   ```bash
   export CLOUDPULSE_ADMIN_EMAIL=...   CLOUDPULSE_ADMIN_PASSWORD=...   # from .env
   python3 tests/e2e/api_workflow.py --restart-cmd "docker compose restart api"
   ```
   It prints PASS/FAIL per step including "data persists after restart".
5. **Browser test (optional):** `cd apps/web && npx playwright install chromium && E2E_EMAIL=... E2E_PASSWORD=... npx playwright test`.
6. Lint: `make lint` (needs `pip install ruff`).
7. **Record your results**: open `docs/TEST_RESULTS.md` and add a section "Run on my machine, <date>" with what actually passed. Only claim what you ran.

---
## STAGE 5 — Use your own data (instead of the demo)

1. Prepare a CSV with this exact header (names are case-sensitive):
   `usage_date,provider,account_identifier,service_name,region,usage_quantity,usage_unit,cost_amount,currency,source_type`
   Example row: `2025-06-01,aws,111122223333,EC2,us-east-1,24.6,Hrs,246.27,USD,manual_csv`.
   Allowed `provider`: aws, azure, other. Allowed `source_type`: manual_csv (use this for your own files), synthetic, aws_cost_explorer, azure_cost_management. Dates are `YYYY-MM-DD`; currency is a 3-letter code.
   AWS source: Billing console → Cost Explorer → export, or a CUR file converted to these columns. Azure: Cost Management → Cost analysis → Download CSV, then rename columns.
2. In the web app: **Cost analytics → Ingestion → Choose file → Upload** (admin only). You see "N inserted, M duplicates, K rejected". Rejected rows are explained with line number and field. Uploading the same file again inserts 0 rows.
3. **Anomalies → Run detection**, then **Cost analytics → Generate forecast** (needs ≥28 days of data; ≥42 for a backtest).
4. Events: send test events with `python3 scripts/send_test_events.py --count 5` (set `CLOUDPULSE_ADMIN_EMAIL` + `CLOUDPULSE_ADMIN_PASSWORD`, or `CLOUDPULSE_INGEST_TOKEN` equal to the server's `INGEST_API_TOKEN`). Real events: POST JSON like `data/sample/sample_event.json` to `/api/v1/events` with header `Authorization: Bearer <INGEST_API_TOKEN>`.
5. Remove demo data when you want a clean slate: `docker compose exec api python -m app.cli reset-dev-data --yes` (development only).

---
## STAGE 6 — Put it on GitHub and make CI work

1. Create an empty repo on GitHub (no README). Locally:
   ```bash
   git init -b main        # skip if the folder already has .git
   git add .
   git status              # confirm .env is NOT listed
   python3 scripts/scan_secrets.py   # must say 0 findings
   git commit -m "Initial import of CloudPulse AI"
   git remote add origin https://github.com/<you>/cloudpulse-ai.git
   git push -u origin main
   ```
2. Open the repo's **Actions** tab. Workflows `CI`, `Security`, `Build images` run on push. Expect some red on the first run — open the failing job, read the log, fix the named file, push again. Typical first-run causes: missing `apps/web/package-lock.json` (commit it), a dependency pin that no longer exists, a lint rule.
3. Turn on protection: Settings → Branches → require pull requests + passing CI on `main`.
4. Optional image publishing: Settings → Secrets and variables → Actions → **Variables** → add `PUBLISH_IMAGES` = `true`. Images go to `ghcr.io/<you>/<repo>/api` and `/web` using the built-in `GITHUB_TOKEN` (no secrets to add). Then make the packages visible: your profile → Packages → each package → Package settings.
5. Replace placeholders `OWNER/REPO` with `<you>/<repo>` in: `gitops/argocd/project.yaml`, `gitops/argocd/application-dev.yaml`, `gitops/argocd/application-prod.yaml`, `gitops/environments/dev/values.yaml`, `gitops/environments/prod/values.yaml`, `infra/helm/cloudpulse/values.yaml`. (Find them: `grep -rn OWNER/REPO .`) Commit and push.

---
## STAGE 7 — Kubernetes (optional): kind, then Argo CD

**7a. Local cluster with kind (free).**
1. Install `kind`, `kubectl`, `helm` (Mac: `brew install kind kubectl helm`; Linux/WSL: see each project's install page).
2. `bash scripts/kind_local.sh up` → builds the two images, creates the cluster, installs a throw-away Postgres and the Helm chart.
3. `kubectl -n cloudpulse port-forward svc/cloudpulse-web 8080:80`, open <http://localhost:8080>.
4. Create the admin inside the cluster: `kubectl -n cloudpulse exec deploy/cloudpulse-api -- env CLOUDPULSE_ADMIN_PASSWORD='Str0ng-Passw0rd-123' python -m app.cli create-admin --email you@example.com`, then `... python -m app.cli load-demo`.
5. Remove: `bash scripts/kind_local.sh down`.
Validate the chart any time with `helm lint infra/helm/cloudpulse` and `helm template cloudpulse infra/helm/cloudpulse | less`.

**7b. Argo CD (GitOps).**
1. In the kind cluster: `kubectl create namespace argocd` and install Argo CD with the manifest from the Argo CD docs (<https://argo-cd.readthedocs.io/en/stable/getting_started/>). Port-forward its server and log in (initial password: `argocd admin initial-password -n argocd`).
2. Create the app's secret (never in Git): 
   ```bash
   kubectl create namespace cloudpulse-dev
   kubectl -n cloudpulse-dev create secret generic cloudpulse-secrets \
     --from-literal=DATABASE_URL='postgresql+psycopg://user:pass@host:5432/cloudpulse' \
     --from-literal=SECRET_KEY="$(openssl rand -base64 48)"
   ```
   You need a reachable PostgreSQL (for a lab: reuse `infra/kubernetes/dev-postgres.yaml` in that namespace).
3. Set the real image tags in `gitops/environments/dev/values.yaml` (the commit SHA of the images CI published), push, then:
   `kubectl apply -f gitops/argocd/project.yaml -f gitops/argocd/application-dev.yaml`
4. Argo CD shows `cloudpulse-dev` syncing. Prod (`application-prod.yaml`) never auto-syncs — you press Sync after reviewing the diff. Rollback = revert the Git commit. Details: `docs/GITOPS_GUIDE.md`.

---
## STAGE 8 — AWS with Terraform (optional, costs money — read carefully)

1. **Prepare the account:** use a non-root IAM user/role (IAM Identity Center is best). Billing → Budgets → create a monthly budget alert (e.g. $10). Install Terraform ≥1.6 and the AWS CLI; `aws configure sso` (or set a profile) and verify `aws sts get-caller-identity`.
2. `bash scripts/package_serverless.sh` → creates `dist/aws_lambda.zip`.
3. ```bash
   cd infra/terraform
   cp envs/dev.tfvars.example envs/dev.tfvars     # edit: region, api_url (leave "" at first)
   terraform init
   terraform fmt -recursive && terraform validate
   terraform plan -var-file=envs/dev.tfvars -out=tfplan
   ```
   `validate` was never run by the author — fix any complaint it prints. **Read the whole plan.** With defaults it creates: an S3 bucket, a DynamoDB table, two Secrets Manager secret *containers*, an SNS topic and a log group (≈ $1–3/month). It will not create the EC2 host or RDS unless you set `enable_app_host` / `enable_rds` to true.
4. Only when you accept the plan: `terraform apply tfplan`.
5. Put real secret values in (outside Terraform): 
   `aws secretsmanager put-secret-value --secret-id $(terraform output -raw ingest_token_secret_arn) --secret-string "$(openssl rand -base64 36)"` and give the **same** token to the API as `INGEST_API_TOKEN` (in `.env`, or the Kubernetes secret).
6. The ingestion Lambda is created only when `api_url` is set to your API's public **https** address (the API must be reachable from the internet; a laptop `localhost` will not work — use the EC2 option or a tunnel). Then upload any cost CSV to `s3://<bucket>/cost-exports/` and watch it appear in the app under Cost analytics → Ingestion.
7. **Destroy when done:** empty the bucket (versioned) then `terraform destroy -var-file=envs/dev.tfvars`. Next day check Billing for stray charges.

---
## STAGE 9 — Connect real AWS / Azure cost data (optional)

**AWS (Cost Explorer):** in the AWS Billing console enable Cost Explorer (data can take up to 24 h). Give the *API process* an identity: on EC2/ECS use an IAM role; on your laptop run with `AWS_PROFILE` set to an SSO profile. Attach the policy printed in `docs/DEPLOYMENT_GUIDE.md` (ce:GetCostAndUsage, sts:GetCallerIdentity). In `.env` set `AWS_REGION=us-east-1`. Install the SDK in the API image: add `boto3` to `apps/api/requirements.txt`, rebuild (`make up`). In the app: **Integrations → AWS → Test connection → Enable → Sync last 30 days**. Compare the totals to the AWS console before trusting them. Note: AWS charges a small fee per Cost Explorer API request.

**Azure:** create a managed identity (or use `az login` locally), grant **Cost Management Reader** + **Reader** on the subscription, set `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` (and `AZURE_CLIENT_ID`) in `.env`, add `azure-identity` to `requirements.txt`, rebuild, then Integrations → Azure → Test → Enable → Sync. Azure Function ingestion: `bash scripts/package_serverless.sh` produces `dist/azure_function.zip`; deploy it with Azure Functions Core Tools or `az functionapp deployment source config-zip`, set app settings `CLOUDPULSE_API_URL`, `INGEST_API_TOKEN` (Key Vault reference) and `RAW_DATA_STORAGE` (storage connection via managed identity), and create a blob container `cost-exports`.

Never paste access keys into the website — it has no field for them on purpose.

---
## STAGE 10 — Monitoring, failure drill, backups

1. `make monitoring` → Prometheus <http://localhost:9090> (Status → Targets should show `cloudpulse-api` UP), Grafana <http://localhost:3001> (user `admin`, password = `GRAFANA_ADMIN_PASSWORD` in `.env`), dashboard "CloudPulse API" in folder CloudPulse.
2. Failure drill: `bash scripts/chaos_local.sh` stops the database, checks `/ready` → 503 with no leaked details and `/health` still 200, restarts the DB and checks recovery. **Local only.**
3. Backup: `bash scripts/backup_db.sh backup` → file in `backups/`. Restore: `bash scripts/backup_db.sh restore backups/<file>`.

---
## STAGE 11 — Optional AI explanations (LLM)
Off by default; the app is complete without it. To enable: in `.env` set `LLM_PROVIDER=anthropic`, `LLM_API_KEY=<your key>`, `LLM_MODEL=<model id from your provider's docs>`, then `make up`. The key stays on the server. Run detection again for new anomalies; the Recommendations page then shows a purple "AI-generated explanation (not a measured fact)" box. If the call fails, the rule-based text is kept. (The provider client has never been tested against the live service.)

---
## STAGE 12 — Make it yours and finish the portfolio

1. Take screenshots (Overview, Cost analytics + forecast, Anomalies, Alert detail, Integrations) and save in `docs/screenshots/`; add them to `README.md` under Features.
2. Add a `LICENSE` file (MIT is common), update the author line in `docs/PROJECT_DOCUMENTATION.md`.
3. In `docs/TEST_RESULTS.md` replace "not executed" statements **only** for things you have now run, with real output.
4. Keep the "Known limitations" honest — recruiters value that.
5. Final checklist before sharing: `python3 scripts/scan_secrets.py` = 0 findings; `.env` not in `git status`; CI green; a clean clone starts with `make setup && make up && make seed`; the demo works with no cloud credentials.

---
## Cheat sheet
| I want to… | Command |
|---|---|
| start / stop | `make up` / `make down` |
| see logs | `make logs` |
| wipe local data | `make reset` |
| create admin + demo data | `make seed` |
| run tests | `make test-unit`, `make test-db`, `make test-web`, `make test-e2e` |
| lint / format | `make lint` / `make fmt` |
| API docs | <http://localhost:8000/api/v1/docs> |
| change detection sensitivity | Anomalies → run via API with `config` (see `docs/API_REFERENCE.md`) |
| regenerate demo CSV | `make sample` |

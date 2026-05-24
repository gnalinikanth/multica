# Deploying Multica on Railway

This guide walks you through deploying the full Multica stack (PostgreSQL, backend API, frontend) on [Railway](https://railway.app).

## How the config files work

`railway.toml` is shared by **every** service that uses this repository root. Because of that:

- **No `[build]` section** — each service must have its own Dockerfile path set in the Railway dashboard. A `[build]` block here would force one Dockerfile on all services.
- **`startCommand` is set here** — `apps/web/package.json` contains a `"start": "next start"` script. Railway auto-detects this and injects `pnpm --filter @multica/web start` as the start command, which fails in the minimal standalone runtime image (no pnpm, no Next.js CLI). Locking `startCommand = "node apps/web/server.js"` in `railway.toml` prevents any dashboard value or auto-detection from overriding the correct command.

## Architecture on Railway

| Service | Dockerfile | Internal hostname |
|---------|-----------|------------------|
| `postgres` | Railway PostgreSQL plugin | `${{Postgres.DATABASE_URL}}` |
| `backend` | `Dockerfile` | `backend.railway.internal:8080` |
| `frontend` | `Dockerfile.web` | public Railway domain / custom domain |

The frontend proxies `/api/*`, `/auth/*`, `/ws`, and `/uploads/*` to the backend through Next.js server-side rewrites — the browser never calls the backend directly.

---

## Prerequisites

- [Railway account](https://railway.app)
- Your fork of this repository pushed to GitHub

---

## Step 1 — Create a Railway project

1. Go to [railway.app/new](https://railway.app/new) → **Empty Project**.
2. Give it a name (e.g. `multica`).

---

## Step 2 — Add PostgreSQL

1. **+ New** → **Database** → **Add PostgreSQL**.
2. Railway provisions a pgvector-enabled PostgreSQL 17 instance and exposes `${{Postgres.DATABASE_URL}}`.

> Multica requires the `pgvector` extension. Railway's PostgreSQL plugin includes it automatically.

---

## Step 3 — Deploy the backend

### 3a — Create the service

1. **+ New** → **GitHub Repo** → select your fork.
2. Railway detects `railway.toml` but **there is no `[build]` section** — you must set the Dockerfile manually:
   - **Service → Settings → Build → Dockerfile Path** → `Dockerfile`
   - **Service → Settings → Build → Build Context** → `/` (repository root)
3. Rename the service to `backend` (used by the frontend's internal URL).

### 3b — Set environment variables

Go to **Service → Variables** and add:

**Required**

| Variable | Value |
|----------|-------|
| `DATABASE_URL` | `${{Postgres.DATABASE_URL}}` (paste this reference exactly) |
| `JWT_SECRET` | `openssl rand -hex 32` output |
| `APP_ENV` | `production` |
| `PORT` | `8080` — pin this so the frontend can reach the backend at `backend.railway.internal:8080` |
| `FRONTEND_ORIGIN` | Frontend public URL (set after Step 4, e.g. `https://multica-web.up.railway.app`) |
| `MULTICA_APP_URL` | Same as `FRONTEND_ORIGIN` |
| `MULTICA_PUBLIC_URL` | Backend public URL, e.g. `https://multica-backend.up.railway.app` |

**Optional**

| Variable | Notes |
|----------|-------|
| `RESEND_API_KEY` | Email delivery (recommended for production) |
| `RESEND_FROM_EMAIL` | Default: `noreply@multica.ai` |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USERNAME` / `SMTP_PASSWORD` | SMTP alternative to Resend |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google OAuth |
| `GOOGLE_REDIRECT_URI` | e.g. `https://multica-web.up.railway.app/auth/callback` |
| `S3_BUCKET` / `S3_REGION` | S3 file uploads (omit to use local disk) |
| `CLOUDFRONT_DOMAIN` / `CLOUDFRONT_KEY_PAIR_ID` / `CLOUDFRONT_PRIVATE_KEY` | CloudFront CDN |
| `CORS_ALLOWED_ORIGINS` | Comma-separated extra allowed origins |
| `ALLOW_SIGNUP` | Default `true`; set `false` to disable registration |
| `ALLOWED_EMAIL_DOMAINS` | Restrict signup to specific domains |
| `MULTICA_DEV_VERIFICATION_CODE` | **Never set in production** |
| `REDIS_URL` | Redis for rate limiting (optional) |
| `GITHUB_APP_SLUG` / `GITHUB_WEBHOOK_SECRET` | GitHub App integration |

### 3c — Set the health check

- **Service → Settings → Deploy → Health Check Path** → `/live`
- **Service → Settings → Deploy → Health Check Timeout** → `300`

The `startCommand` from `railway.toml` (`node apps/web/server.js`) is passed as an argument to the backend's `ENTRYPOINT` (`./entrypoint.sh`). The entrypoint script ignores all arguments and runs its fixed sequence: database migrations → Go server. No action needed on your part.

### 3d — Deploy

Click **Deploy**. The entrypoint runs migrations before starting the server. If PostgreSQL is still initialising, the container exits and Railway retries automatically (up to 10 times per `railway.toml`).

---

## Step 4 — Deploy the frontend

`Dockerfile.web` requires the full monorepo as its build context (it copies `packages/`, `pnpm-workspace.yaml`, etc.). Both services therefore use `/` as their Railway root directory.

### 4a — Create the service

1. **+ New** → **GitHub Repo** → select the **same fork**.
2. Set the Dockerfile manually — this step is critical:
   - **Service → Settings → Build → Dockerfile Path** → `Dockerfile.web`
   - **Service → Settings → Build → Build Context** → `/` (repository root)
3. Rename the service to `frontend`.

### 4b — Set environment variables

> **`REMOTE_API_URL` is a Docker build argument**, not just a runtime variable. It is baked into the Next.js rewrite rules at build time. It **must** be set before the first deploy, and the service must be redeployed whenever it changes.

**Required**

| Variable | Value |
|----------|-------|
| `REMOTE_API_URL` | `http://backend.railway.internal:8080` — Railway private network URL for the backend. The default in `Dockerfile.web` (`http://backend:8080`) is a Docker Compose hostname and **will not work on Railway**. |
| `NEXT_PUBLIC_WS_URL` | Backend **public** WebSocket URL, e.g. `wss://multica-backend.up.railway.app` |

**Optional**

| Variable | Notes |
|----------|-------|
| `NEXT_PUBLIC_APP_VERSION` | Version string shown in UI (default: `dev`) |
| `POSTHOG_API_KEY` / `POSTHOG_HOST` | Product analytics |

### 4c — No health check or start command needed

- The `startCommand` in `railway.toml` (`node apps/web/server.js`) is already correct for the frontend.
- Do **not** set a health check path — Railway will consider the service healthy once it is listening on `$PORT`. The Next.js standalone server always responds with `200` on `/`.

### 4d — Deploy

Click **Deploy**. Railway passes `REMOTE_API_URL` as a Docker build argument and the Next.js standalone server starts on Railway's injected `$PORT`.

---

## Step 5 — Wire up cross-service URLs

After both services have public Railway domains:

1. **Backend → Variables** → set `FRONTEND_ORIGIN` and `MULTICA_APP_URL` to the frontend's domain.
2. **Frontend → Variables** → confirm `REMOTE_API_URL` is `http://backend.railway.internal:8080` and `NEXT_PUBLIC_WS_URL` is the backend's `wss://` domain.
3. Redeploy both services for the updated variables to take effect.

---

## Internal networking

Services in the same Railway project communicate over a private network:

```
http://<service-name>.railway.internal:<PORT>
```

The backend is pinned to port `8080` (via `PORT=8080` in its variables), so the frontend always reaches it at:

```
http://backend.railway.internal:8080
```

If you rename the backend service, update `REMOTE_API_URL` in the frontend to match.

---

## Custom domains

1. **Service → Settings → Networking → Generate Domain** for a Railway subdomain, or **Add Custom Domain**.
2. Update `FRONTEND_ORIGIN`, `MULTICA_APP_URL`, and `MULTICA_PUBLIC_URL` in the backend.
3. Update `NEXT_PUBLIC_WS_URL` in the frontend (use `wss://` for production).

---

## Common pitfalls

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Frontend container exits: `pnpm: not found` | Railway auto-detected `pnpm --filter @multica/web start` from `apps/web/package.json` and set it as the dashboard start command, overriding the Dockerfile CMD | `railway.toml` now locks `startCommand = "node apps/web/server.js"`. If you see this error, re-pull the latest `railway.toml` and redeploy; also clear any custom start command in the dashboard. |
| Frontend builds but API calls return 404 / CORS errors | `REMOTE_API_URL` was not set before the build, so Next.js rewrites compiled with the wrong backend URL | Set `REMOTE_API_URL=http://backend.railway.internal:8080` in the frontend's variables and **redeploy** (a restart is not enough — the Next.js build must rerun). |
| Backend shows wrong Dockerfile (Node.js build for Go service) | A `[build]` block existed in `railway.toml` and forced `Dockerfile.web` on all services | `railway.toml` no longer has a `[build]` section. Confirm **Service → Settings → Build → Dockerfile Path** is set to `Dockerfile` for the backend and `Dockerfile.web` for the frontend. |
| `migrate: connection refused` on backend startup | Railway deploys services in parallel; PostgreSQL may not be ready yet | The `on_failure` restart policy retries up to 10 times. Wait for PostgreSQL to show healthy in the Railway dashboard, then trigger a manual redeploy if needed. |
| WebSocket connection drops immediately | `NEXT_PUBLIC_WS_URL` is `ws://` (insecure) on a public Railway domain | Set `NEXT_PUBLIC_WS_URL` to `wss://multica-backend.up.railway.app` and redeploy the frontend. |
| Email verification never arrives | `RESEND_API_KEY` or SMTP variables not set | Configure email variables. For local testing only: set `MULTICA_DEV_VERIFICATION_CODE=888888` and `APP_ENV=development` — **never on a public instance**. |
| `pnpm install --frozen-lockfile` fails during build | `pnpm-lock.yaml` is out of sync with `package.json` | Run `pnpm install` locally to regenerate the lockfile, commit it, and redeploy. |

---

## One-click deploy (optional)

After forking, add this badge to your README:

```markdown
[![Deploy on Railway](https://railway.app/button.svg)](https://railway.app/template/new?template=https://github.com/<your-org>/multica)
```

Replace `<your-org>` with your GitHub username or organization.

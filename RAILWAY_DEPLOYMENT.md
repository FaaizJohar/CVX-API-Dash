# OmniRoute Railway Deployment Guide

This guide describes how to deploy OmniRoute to Railway for production using its existing Dockerfile and configuration.

## Prerequisites

- A GitHub repository with this codebase.
- A Railway account (https://railway.app).
- Basic understanding of environment variables and volume mounts in Railway.

## Recommended Deployment Method

OmniRoute handles its own build process via a robust, multi-stage `Dockerfile`.
Railway automatically supports Docker deployments. We also provide a `railway.json` file which instructs Railway to use the Dockerfile and configure a proper health check (`/healthz`).

## How to Deploy from GitHub

1. Create a new project in Railway.
2. Select **Deploy from GitHub repo**.
3. Connect your repository and select the OmniRoute repository.
4. Railway will detect the `Dockerfile` (assisted by `railway.json`) and start building.

### Critical: Configure Persistent Storage

OmniRoute uses SQLite for all its state (API keys, logs, provider configurations, sessions). **Railway containers are ephemeral.** You must mount a persistent volume so that your database survives redeployments.

1. Go to your **Project** -> **Service** (omniroute).
2. Navigate to the **Volumes** tab.
3. Create a new volume (e.g., `omniroute-data`).
4. Mount it at the path: `/app/data` (this path matches the `DATA_DIR` environment variable preset in the Dockerfile).

If you change the mount path in Railway, ensure you set the `DATA_DIR` environment variable in Railway to match your new mount path.

### Environment Variables

Before your app becomes fully functional, go to the **Variables** tab of your service and configure the necessary secrets.

**Required Secrets:**

- `JWT_SECRET`: Used for signing session cookies. Generate one via `openssl rand -base64 48`.
- `API_KEY_SECRET`: Used to encrypt your API keys at rest. Generate one via `openssl rand -hex 32`.
- `INITIAL_PASSWORD`: Sets the default administrator password. It is highly recommended to set this to a strong password.

**Other Notable Variables:**

- `REQUIRE_API_KEY`: Set to `true` (default in the image) for production deployments so random users on the internet can't access your LLMs for free.
- `DATA_DIR`: Set to `/app/data` by default. You do not need to override this unless you mount the Railway volume elsewhere.
- `STORAGE_ENCRYPTION_KEY`: (Optional) Encrypt the entire SQLite database at rest.

The container listens on `PORT`, which is seamlessly injected and overridden by Railway.

### Redis Setup (Optional)

OmniRoute has a built-in memory-based rate limiter. Redis is completely **optional**. Do not provision Redis unless you specifically plan to run multiple replicas of OmniRoute that need to share rate limit state.

If you do need Redis:

1. Provision a Redis service in your Railway project.
2. In the OmniRoute service variables, set `REDIS_URL` to the private connection URL of the Redis service provided by Railway.

## Networking and Public Domain

1. In Railway, navigate to the **Settings** tab of your service.
2. Scroll down to **Networking**.
3. Click **Generate Domain** to get a free Railway subdomain, or **Custom Domain** to configure your own.
4. Set the `NEXT_PUBLIC_BASE_URL` environment variable to match your newly generated public domain (e.g., `https://omniroute-production.up.railway.app`). This is essential for proper OAuth callbacks and links in the dashboard.

## Health Checks

The `railway.json` file configures Railway to poll the `/healthz` endpoint on startup to verify the app is ready.
OmniRoute includes a specific lightweight lifecycle endpoint that prevents timeout failures from deep database reads during active streaming sessions.
The health check ensures the container will only accept traffic when fully initialized.

## Validation and Post-Deployment Checklist

- [ ] I mounted a Volume to `/app/data`.
- [ ] I set `JWT_SECRET`, `API_KEY_SECRET`, and `INITIAL_PASSWORD`.
- [ ] I verified the deployment is green and Railway routed traffic.
- [ ] I navigated to the public domain and was prompted to log in.
- [ ] I logged in with `admin` and `INITIAL_PASSWORD`.
- [ ] I checked that restarting the container in Railway does NOT clear my data.

## Backups and Recovery

Railway automatically handles volume backups depending on your plan, but you can also configure OmniRoute's `DISABLE_SQLITE_AUTO_BACKUP` if you prefer to rely entirely on Railway's volume snapshots. For production, manual database exports via the OmniRoute dashboard are recommended before major upgrades.

## Updating and Rollbacks

Pushing to your GitHub repository triggers an automatic Railway redeployment. OmniRoute's SQLite migrations are idempotent; they will automatically upgrade the schema on boot.

If a deployment fails, Railway will not switch traffic to it. You can rollback to an older deployment directly from the Railway dashboard by finding the successful build history and clicking "Redeploy".

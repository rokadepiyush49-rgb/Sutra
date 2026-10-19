# Deploying SUTRA

SUTRA ships as two container images plus PostgreSQL. This guide covers what has to run, the
configuration it needs, and three ways to host it, from a single server to the full AWS platform.

| Piece | Image / service | Port | Health check |
|---|---|---|---|
| API and batch jobs | `docker/api.Dockerfile` (models are trained during the build) | 8000 | `/health/live`, `/health/ready` |
| Web app and session BFF | `docker/web.Dockerfile` (Next.js standalone, distroless) | 3000 | `/api/health` |
| Database | PostgreSQL 17 | 5432 | `pg_isready` |
| Rate-limit store (optional) | Valkey / Redis | 6379 | `PING` |

One-off jobs run from the API image:

| Job | Command | When |
|---|---|---|
| Migrate | `alembic upgrade head` (working directory `/app/backend`, the image default) | Every release, before the API starts |
| Seed demo data + detect | `python -m app.cli demo` | First start of a demo environment. It **truncates** the data tables, so never run it against data you want to keep |
| Detect only | `python -m app.cli detect` | After new data arrives (the Helm chart runs it as a CronJob) |

## Before you deploy

- **HTTPS is required.** In production builds the session cookies are `Secure`, and the web app
  sends HSTS. Browsers drop `Secure` cookies on plain `http://` (except `localhost`), so sign-in
  would loop. Put the web app behind TLS: a platform load balancer or a reverse proxy.
- **Only the web app is public.** The browser never calls the API directly; the web app proxies
  every request with the JWT it keeps in an `httpOnly` cookie. Keep port 8000 private.
- **Memory.** Building the API image trains the models on 2 parallel processes of about 1.5 GB
  each. We build it in an 8 GB Docker VM; on a smaller builder pass `--build-arg TRAIN_WORKERS=1`.
  At rest the API uses about 200 MB, but seeding and detection peaked at 2.6 GB in our run, and
  Live Ops runs detection inside the API process, so give the API at least 3 GB and the host 4 GB.
- **Demo mode is public.** With `SUTRA_DEMO_MODE=true`, anyone who can open the site can sign in
  as any demo persona, and the mock bank (`/bank`) is enabled. That is fine for the synthetic
  Kestrel-Sim data and nothing else. `SUTRA_ENV=staging|prod` refuses to start in demo mode.

## Configuration

Generate each secret with `python -c "import secrets; print(secrets.token_urlsafe(48))"`.

### API (and the migrate / seed / detect jobs)

| Variable | Required | Notes |
|---|---|---|
| `SUTRA_ENV` | yes | `dev` for a demo. `staging` / `prod` refuse default secrets, demo mode, and (prod) non-S3 evidence storage |
| `SUTRA_DATABASE_URL` | yes | `postgresql+psycopg://user:password@host:5432/sutra` (add `?sslmode=require` for managed databases) |
| `SUTRA_JWT_SECRET` | yes | Random, 32+ characters |
| `SUTRA_PII_HMAC_KEY` | yes | Random, 32+ characters. Changing it breaks links between already-tokenised records |
| `SUTRA_DEMO_MODE` | yes | `true` for a public demo, `false` otherwise |
| `SUTRA_DEMO_PASSWORD` | yes | Password given to the seeded users. With demo mode off, people sign in with a username and this password |
| `SUTRA_STORAGE_BACKEND` | no | `local` (default) or `s3`. Containers run read-only, so set `SUTRA_STORAGE_DIR=/tmp/packs` with `local` |
| `SUTRA_S3_BUCKET`, `SUTRA_S3_KMS_KEY_ID`, `SUTRA_AWS_REGION` | with `s3` | Evidence packs in S3 Object Lock |
| `SUTRA_RATE_LIMIT_STORAGE_URI` | with 2+ replicas | `redis://host:6379/0`. The default `memory://` is per process |
| `SUTRA_CORS_ORIGINS` | no | JSON list. Only matters if a browser calls the API directly, which the web app never does |
| `SUTRA_LOG_JSON` | no | `true` in the image |
| `ANTHROPIC_API_KEY` | no | Ask SUTRA answers with Claude. Without it, deterministic templates answer |
| `GROQ_API_KEY` | no | Investigation Council agents. Without it, the deterministic council runs |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | no | Send traces to an OpenTelemetry collector |

Every setting, with defaults, is in [`.env.example`](../../.env.example) and
[`backend/app/core/config.py`](../../backend/app/core/config.py).

### Web app

| Variable | Required | Notes |
|---|---|---|
| `SUTRA_API_URL` | yes | Private URL of the API, e.g. `http://api:8000` |
| `SUTRA_DEMO_MODE` | yes | Must match the API. `true` shows the persona picker |
| `SUTRA_DEMO_PASSWORD` | in demo mode | Must match the API. Read on the server only; never sent to the browser |
| `PORT`, `HOSTNAME` | no | Default `3000` and `0.0.0.0` in the image |

## Option A: one Linux server with Docker Compose

The quickest public demo. [`docker-compose.yml`](../../docker-compose.yml) already starts the stack
in order (postgres → migrate → seed → api → web) with health checks, read-only containers and all
capabilities dropped. Every port is bound to `127.0.0.1`, so nothing is reachable from outside until
you add a TLS proxy.

Verified on macOS (Docker Desktop, 8 GB VM): the full stack comes up healthy, the seed job passes
14/14 scenario checks, and sign-in, cases and evidence packs work through the web app.

**On Azure for Students**, follow [azure-vm.md](azure-vm.md): every portal setting and server command
for a `Standard_B2as_v2` VM in Central India, with a free `*.cloudapp.azure.com` address and HTTPS.

1. A server with Docker Engine and the Compose plugin, at least 4 GB of RAM (8 GB to build the
   images on it), and a DNS name pointing at it.
2. Clone the repository and create `.env` next to `docker-compose.yml` with your own secrets:

   ```bash
   SUTRA_JWT_SECRET=<random 48 characters>
   SUTRA_PII_HMAC_KEY=<random 48 characters>
   SUTRA_DEMO_PASSWORD=<anything; demo sign-in never shows it>
   ```

3. Start it:

   ```bash
   docker compose up -d --build
   docker compose ps          # migrate and seed: Exited (0); postgres, valkey, api, web: healthy
   ```

4. Put HTTPS in front of `127.0.0.1:3000`. With [Caddy](https://caddyserver.com/docs/install)
   installed on the host, this `Caddyfile` gets and renews a certificate automatically
   (not yet tried against a live domain):

   ```caddyfile
   sutra.example.com {
       reverse_proxy 127.0.0.1:3000
   }
   ```

   Open only ports 80 and 443 in the firewall.

5. Check it: `https://sutra.example.com/api/health` returns `{"status":"ok"}`, and the sign-in page
   lists the demo personas.

Note that the `seed` service re-seeds the demo world every time the stack starts, which resets
cases, notes and Live Ops runs. To keep data across restarts, start only the long-running services
after the first run: `docker compose up -d --no-deps postgres valkey api web`.

## Option B: a managed container platform

Any platform that runs Docker images with private networking and managed PostgreSQL works
(for example Render, Railway, Fly.io, Google Cloud Run or AWS App Runner). Not yet tried on a
specific platform; the shape is the same everywhere:

1. Create PostgreSQL 17 and note its connection string.
2. Build and push both images (or let the platform build from `docker/api.Dockerfile` and
   `docker/web.Dockerfile` with the repository root as the build context).
3. Create the **API** service from the API image: private only, port 8000, readiness
   `/health/ready`, liveness `/health/live`, and the API variables above.
4. Run the **migrate** job, then the **seed** job (`python -m app.cli demo`) once, from the API image
   with the same variables.
5. Create the **web** service from the web image: public, port 3000, health `/api/health`,
   `SUTRA_API_URL` set to the API's private URL. The platform's HTTPS endpoint covers the TLS
   requirement.

## Option C: AWS (EKS, Helm, Argo CD)

The production design: Terraform in [`infra/terraform`](../../infra/terraform), the Helm chart in
[`deploy/helm/sutra`](../../deploy/helm/sutra), GitOps in [`deploy/argocd`](../../deploy/argocd), and
the release pipeline in [`.github/workflows/release.yml`](../../.github/workflows/release.yml), which
stays skipped until the `AWS_RELEASE_ROLE_ARN` repository variable is set. Step-by-step setup,
releases, rollback and restore are in the [runbook](../operations/runbook.md); costs are in
[cost.md](../operations/cost.md). CI validates all of it offline (`terraform validate`, tflint,
Checkov, `helm lint`, kubeconform); applying it needs an AWS account.

## After every deploy

```bash
curl -fsS https://<your-host>/api/health                 # web
docker compose exec api python -c "import urllib.request as u; print(u.urlopen('http://127.0.0.1:8000/health/ready').read().decode())"
```

`/health/ready` returns 503 with the failing check named (`database` or a model) until Postgres
answers and the three models load. Then sign in and open the top case.

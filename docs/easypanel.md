# Deploying on Easypanel

Easypanel builds and runs the repo's `Dockerfile` directly — it does
**not** use `docker-compose.yml`, so Compose-only settings (like the
`HOST_PORT` mapping) don't apply here. The image itself needs nothing
Easypanel-specific; this page just walks through the App settings.

## 1. Create the App

- **Source**: point Easypanel's Git source at your fork, branch `main`
  (or whichever branch you deploy from).
- **Build method**: `Dockerfile` (auto-detected — the repo has one at
  the root).

## 2. Build Args (`NEXT_PUBLIC_*`)

These are inlined into the client bundle at build time, so they must
be set as **Build Arguments** in Easypanel, not runtime env vars —
setting them only as env vars will build a bundle pointing at nothing:

| Build arg                      | Required |
| ------------------------------- | -------- |
| `NEXT_PUBLIC_SUPABASE_URL`      | yes      |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | yes      |
| `NEXT_PUBLIC_SITE_URL`          | recommended — your Easypanel domain, e.g. `https://crm.example.com` |
| `NEXT_PUBLIC_APP_LOCALE`        | optional, defaults to `en` |

Changing any of these later requires a rebuild ("Deploy" in Easypanel
triggers a fresh build, so this happens automatically — there's no
extra step).

## 3. Environment Variables (runtime secrets)

Set these as regular **Environment Variables** (not build args) —
they're read at container start and are never baked into the image:

- `SUPABASE_SERVICE_ROLE_KEY`
- `ENCRYPTION_KEY`
- `META_APP_SECRET`
- any optional vars you need from `.env.local.example`
  (`AUTOMATION_CRON_SECRET`, `META_APP_ID`, `ALLOWED_INVITE_HOSTS`, …)

See `.env.local.example` for what each one does and which are
required vs. optional.

## 4. Port and domain

- **Port**: `3000` — matches the `EXPOSE 3000` / `PORT=3000` baked
  into the runner stage. Don't override `PORT` in the environment
  variables panel; leave it unset and let the image's default stand.
- **Domain**: attach your domain and enable HTTPS in Easypanel as
  usual. Whatever hostname you assign here should match
  `NEXT_PUBLIC_SITE_URL` above.
- **Health check**: the image ships a Docker `HEALTHCHECK` (polls
  `/` on `:3000`), so Easypanel's own health monitoring picks it up
  without extra config.

## Notes (same as plain Docker)

- Database migrations under `supabase/` are **not** run by the
  container — apply them with the Supabase CLI as described in the
  README before first deploy, and after pulling upstream changes.
- Nothing inside the container is scheduled. If you use automation
  Wait steps or flows, point an external scheduler (e.g. Easypanel's
  own cron, or any external one) at `GET /api/automations/cron` and
  `GET /api/flows/cron` on this deployment, sending the shared secret
  in the `x-cron-secret` header (`AUTOMATION_CRON_SECRET`). Both
  return 503 until that variable is set.
- See [docs/docker.md](./docker.md) for the general Docker build/run
  notes (build-arg vs. env var split, plain `docker run` invocation)
  that apply regardless of platform.

# Twenty CRM — homelab deploy (CT 214)

Self-hosted Twenty for **alpina-tech**, public at `https://crm.alpina.solutions`,
gated by Twenty's own workspace auth. Agents reach it via the REST/GraphQL API and the
`alpina-tech/twenty-mcp` MCP server.

- **Host:** CT 214 (`192.168.31.214`), standalone-docker LXC on PVE `192.168.31.182`.
- **Stack:** `server` + `worker` + `db` (Postgres 16) + `redis`, all in this compose.
- **Compose:** `docker-compose.yml` here is vendored from
  `packages/twenty-docker/docker-compose.yml`. Re-sync after upstream merges.

## Deploy

```sh
# on CT 214
mkdir -p /opt/twenty && cd /opt/twenty
# copy this docker-compose.yml + a filled .env (see .env.example) here
docker compose pull
docker compose up -d
curl -fsS http://localhost:3000/healthz   # -> ok (first boot runs migrations, ~1-2 min)
```

Secrets are generated on the CT and mirrored to PVE `/root/.homelab-creds/twenty.txt`
(chmod 600): `ENCRYPTION_KEY`, `PG_DATABASE_PASSWORD`, plus `TWENTY_API_KEY` /
`TWENTY_API_URL` after the first API key is created.

## First run

1. Open `http://192.168.31.214:3000/` on the LAN, create the workspace + admin user.
2. Settings → Developers → create an API key (used by twenty-mcp and curl).
3. Disable public sign-up in the workspace admin settings (this version has no
   `IS_SIGN_UP_ENABLED` env flag — it is a UI toggle). Invite teammates from inside.

## Public exposure

DNS + tunnel live in the **alpina** Cloudflare account. The ingress rule
`crm.alpina.solutions → http://192.168.31.214:3000` must precede any wildcard rule.
See the infra repo plan `docs/superpowers/plans/2026-06-01-twenty-crm-selfhost.md`.

## Custom images (phase 4)

CI (`.github/workflows/build-images.yml`) builds server/worker images and pushes to
`registry.aisuffer.com/twenty:<sha>` (homelab Registry, CT 209). Point `TAG` /
image at the registry tag, `docker login registry.aisuffer.com`, then
`docker compose pull && docker compose up -d`.

## Backup

PVE-host cron `twenty-backup.sh` runs `pg_dump` + tar of `server-local-data` →
`/mnt/nas/twenty/` (14d retention), included in restic on CT 213. The historical
`twenty-2026-05-14-*.sql.gz` dump is preserved, not rotated.

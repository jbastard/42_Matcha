# Hosting and release

How Matcha runs on the OVH VPS (Debian 13, 4 vCPU, 7.6 GiB RAM, Docker + Compose, shared with Blackjack and a Valheim server) and how a pushed tag becomes a deployment. The edge and serving model follow ADR 0007; the process model follows ADR 0005; migrations follow ADR 0003. Steps only a human can perform are delivered later as a wizard (`mattpocock-skills:wizard`).

## Topology

- **`edge` stack** (`/opt/edge`, root): official Caddy 2 image publishing 80/tcp, 443/tcp and 443/udp, persistent `/data` volume (certificates, never treated as a cache), external Docker network `edge`. One site block per app:
  - `matcha.dishaah.fr`: `reverse_proxy matcha-app:3000`
  - `gamble.dishaah.fr`: `reverse_proxy blackjack-web:80`
  - Compression and the automatic HTTP to HTTPS redirect only. Files versioned under `deploy/edge/`.
- **`matcha` stack** (`/opt/matcha`, user `matcha`): publishes no port.
  - `app`: `ghcr.io/jbastard/matcha-app:X.Y.Z`, one Node process serving `/api`, `/ws` and the SPA build (R-GEN-07 built-in server). On networks `edge` (alias `matcha-app`) and the stack's default network. Health check on `GET /health`. Security headers set by Express.
  - `migrate`: same image, runs the forward-only migrations and exits; `app` depends on it with `service_completed_successfully`.
  - `db`: `postgis/postgis` pinned to PostgreSQL 18, default network only.
- **Blackjack**: unchanged images except its web Caddy listening on `:80` (plain HTTP), no published port, joined to `edge`.
- **DNS** (OVH): `matcha.dishaah.fr` A `151.80.233.236` and AAAA `2001:41d0:305:2100::7a51`.
- **Reserved for the calls bonus**: 3478 and 5349 (tcp/udp) and a UDP relay range such as 49160-49200, for a host-networked coturn decided in the bonus spec.

## Files and data on the VPS

```
/opt/matcha/
  deploy.sh          from deploy/deploy.sh, target of the forced deploy key
  compose.yml        extracted from the deployed image
  .env               production secrets, chmod 600, owner matcha (R-GEN-11)
  data/postgres/     bind mount, database files
  data/uploads/      bind mount, re-encoded photos
  data/backups/      daily pg_dump and pre-deploy dumps
```

- The repo holds `.env.example` with names only. The `.env` also holds `IMAGE_TAG` and `APP_UID`/`APP_GID`, rewritten by `deploy.sh` for the tag.
- Memory: a 2 GiB swap file on the host; `mem_limit` of 1 GiB for `db`, 512 MiB for `app`, 256 MiB for `migrate`.
- Logging: json-file, 10 MB x 5, as Blackjack.

## Release flow

1. A `vX.Y.Z` tag is pushed. The workflow (`on: push: tags: ['v*']`, one deploy at a time via `concurrency`) runs the CI checks, builds `matcha-app` and pushes it to GHCR as `X.Y.Z`. The image embeds the production `compose.yml` at `/deploy/compose.yml`.
2. The deploy job, in a GitHub Environment `production` restricted to `v*` tags, SSHes as `matcha` with a key forced to `/opt/matcha/deploy.sh` and sends `deploy vX.Y.Z`.
3. `deploy.sh` (same shape as Blackjack's, accepts only `deploy <vX.Y.Z>` and `status`):
   - pulls the image and extracts `compose.yml`, keeping the previous one;
   - writes a `pg_dump` to `data/backups/pre-<tag>.sql.gz`;
   - sets `IMAGE_TAG` and runs `docker compose up -d --remove-orphans --wait`;
   - on failure (migration error or failed health check) restores the previous `compose.yml` and tag and starts again; then prunes images.

## Rollback

- Rolling back is redeploying the previous tag. Migrations are never reversed: each one stays compatible with the previous image (expand, then contract in a later release).
- A migration that damages data is repaired by hand from the pre-deploy dump.

## Backups

- Daily `pg_dump` (user `matcha` timer) into `data/backups`, kept 7 days.
- A second restricted SSH key pulls `data/backups` and `data/uploads` off the host with `rrsync` in read-only mode.

## Secrets and access

- User `matcha` in the `docker` group, which is root-equivalent; accepted as for `blackjack`.
- GHCR packages are public (the repository is), so the VPS needs no registry token.
- GitHub holds only the deploy key, host, user and `known_hosts`, scoped to the `production` Environment.

## Human-only steps (wizard input)

1. Release BlackjackOnline with its web service on plain HTTP, no published port, joined to the external `edge` network.
2. With sudo: create the `edge` network, install `/opt/edge` from `deploy/edge/`, stop Blackjack's web, start the edge, redeploy Blackjack, check `gamble.dishaah.fr`.
3. Create the 2 GiB swap file.
4. Create user `matcha` (docker group), `/opt/matcha` and `data/` directories, install `deploy.sh`.
5. Generate the deploy key (forced command) and the backup key (`rrsync -ro`); store the deploy key, host, user and `known_hosts` in the `production` Environment.
6. Write `/opt/matcha/.env` from `.env.example`.
7. Add the `matcha.dishaah.fr` A and AAAA records.
8. Push the first tag; make the GHCR package public after its first publish; check the certificate and `/health`.
9. Install the daily backup timer; seed the 500+ profiles (R-GEN-06).

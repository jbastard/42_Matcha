# HTTPS Docker deployment on a VPS with tag-triggered CI

Research for issue #10. Question: what is the simplest robust way to deploy a Docker Compose app publicly over HTTPS on a single VPS, triggered by pushing a git tag? Covers reverse proxies with automatic TLS, GitHub Actions deploy patterns, secret handling on the VPS, WebSocket proxying and rollback.

Sources were fetched on 2026-11-24. Claims marked **[unverified]** could not be confirmed against a primary source.

## Short answer

- **Reverse proxy:** Caddy, as a Compose service owning ports 80 and 443. Automatic HTTPS, HTTP to HTTPS redirect, renewal and WebSocket proxying work with no extra config. The only condition is persisting its `/data` volume. See the subject-rule caveat below (R-GEN-04, R-GEN-07): nginx is the fallback if Caddy is judged not allowed.
- **CI:** a workflow triggered by `push: tags: ['v*']` builds the images, pushes them to GHCR tagged with the semver version, then SSHes to the VPS to run `docker compose pull && docker compose up -d --wait`.
- **Secrets:** the production `.env` exists only on the VPS, created by a human, never in Git or in an image (R-GEN-11). GitHub holds only the deploy SSH key, host and known_hosts.
- **Rollback:** set the image tag in the VPS `.env` back to the previous version and run `docker compose up -d`. Database migrations are not rolled back by this; keep them backward compatible.

## Subject rules involved

- **R-GEN-11:** credentials, API keys and environment variables live in a local `.env` file excluded from Git. Public credentials can fail the project. This drives the secret design: the VPS keeps its own `.env`, the repo commits only a `.env.example` without values.
- **R-GEN-12:** only what is in the Git repository is evaluated. The Compose file, proxy config and workflow must live in the repo, so the deployment is reproducible from it.
- **R-GEN-07:** "Web server of our choice (Apache, Nginx, built-in server)". Combined with **R-GEN-04** (anything not explicitly allowed is forbidden), a strict reading could consider Caddy or Traefik outside the list. The parenthesis reads as examples, but this is an interpretation, not verified with the 42 staff. The decision ticket should settle it. If in doubt: nginx in front, or Caddy only as a TLS terminator in production while the app itself is served by Node's built-in server (which is explicitly allowed).
- **R-CHAT-01, R-NOTI-01:** real-time delivery within 10 seconds, so the proxy must carry WebSocket (or SSE) connections.

## 1. Reverse proxy with automatic TLS

### Caddy

- Automatic HTTPS requires that the domain's A/AAAA records point to the server, ports 80 and 443 are open externally, and the data directory is writable and persistent. https://caddyserver.com/docs/automatic-https
- Let's Encrypt and ZeroSSL are both enabled by default. If one fails, Caddy tries the other, then backs off and retries (up to 1 day between attempts, for up to 30 days). https://caddyserver.com/docs/automatic-https
- HTTP is redirected to HTTPS on port 80, and certificates are obtained and renewed automatically. https://caddyserver.com/docs/automatic-https
- Minimal config is the domain name followed by `reverse_proxy <upstream>`. https://caddyserver.com/docs/quick-starts/reverse-proxy
- WebSocket: the `reverse_proxy` docs state the proxy supports WebSocket connections, with no option needed to enable it. It sets `X-Forwarded-For`, `X-Forwarded-Proto` and `X-Forwarded-Host` by default and passes `Host` through. https://caddyserver.com/docs/caddyfile/directives/reverse_proxy
- Docker: `/data` holds certificates and private keys and "must not be treated as a cache". `/config` does not need to be persisted. The official image page gives a Compose example publishing `80:80`, `443:443` and `443:443/udp` with named volumes `caddy_data` and `caddy_config`. https://hub.docker.com/_/caddy

Example sketch (adapted, not tested):

```caddyfile
{$DOMAIN} {
	reverse_proxy /api/* api:3000
	reverse_proxy /ws/* api:3000
	reverse_proxy web:80
}
```

### Traefik (v3.7 docs)

- ACME is configured through a `certificatesResolvers` entry with `email`, `storage` (default `acme.json`) and `httpChallenge.entryPoint`, which must be reachable by Let's Encrypt on port 80. Domains are inferred from router `Host()` rules. https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/
- The Docker provider reads container labels through the Docker API. The docs warn that accessing the Docker API without restriction is a security concern, and `exposedByDefault` is `true` unless changed. https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- The `acme.json` permission requirement (600) is commonly cited but was not found on the fetched page: **[unverified]**.
- WebSocket support without config: **[unverified]** (not checked on a primary page).

Trade-off: label-driven config is useful with many services; for one app with two or three services it adds a socket mount and more config than Caddy.

### nginx + certbot

- nginx does not forward `Upgrade` and `Connection` (hop-by-hop headers), so WebSocket locations need `proxy_set_header Upgrade $http_upgrade;` and `proxy_set_header Connection "upgrade";` (or a `map` for mixed traffic), plus `proxy_http_version 1.1` before nginx 1.29.7. Idle connections close after 60 s unless `proxy_read_timeout` is raised or the backend sends pings. https://nginx.org/en/docs/http/websocket.html
- Certbot's own docs advise against Docker unless "you are sure you know what you are doing", and say the Docker mode "is unable to install certificates or configure your webserver". Renewal must be wired by hand (webroot shared with nginx plus a scheduled `certbot renew` and an nginx reload). https://eff-certbot.readthedocs.io/en/latest/install.html
- The renew-and-reload container pattern comes from community sources, not EFF docs: **[unverified]** as an official pattern.

### Comparison

| | Caddy | Traefik | nginx + certbot |
|---|---|---|---|
| TLS issuance and renewal | built in | built in (resolver config) | separate tool, manual wiring |
| HTTP to HTTPS redirect | automatic | entrypoint config | manual server block |
| WebSocket | automatic | [unverified] | explicit headers and timeout |
| Docker socket needed | no | yes (Docker provider) | no |
| Named in R-GEN-07 | no | no | yes |

### Let's Encrypt limits

- 5 certificates per exact set of identifiers every 7 days, 50 per registered domain every 7 days. Use the staging environment while testing. https://letsencrypt.org/docs/rate-limits/
- Consequence: losing Caddy's `/data` volume and recreating it repeatedly can hit the duplicate-certificate limit.

## 2. GitHub Actions deploy patterns

### Trigger

- `on: push: tags:` matches patterns against the ref name. Docs show `v2` and `v1.*`; `v*` follows the same glob syntax. With only `tags`, branch pushes do not trigger. https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions
- `concurrency` keeps one deploy at a time; group names may use the `github` context. https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions

### Option A: build in CI, push to GHCR, pull on the VPS (recommended)

- Official GitHub example: `permissions: packages: write`, `docker/login-action` with `registry: ghcr.io`, `username: ${{ github.actor }}`, `password: ${{ secrets.GITHUB_TOKEN }}`, then `docker/metadata-action` and `docker/build-push-action`, optionally `actions/attest`. https://docs.github.com/en/actions/publishing-packages/publishing-docker-images
- `docker/metadata-action` tags: `type=semver,pattern={{version}}` turns `v1.2.3` into `1.2.3`; `latest` is added automatically on tag pushes (`flavor: latest=auto`). Examples use `@v6`. https://github.com/docker/metadata-action
- Images pushed from a workflow with `GITHUB_TOKEN` are linked to the repo. First publish is **private** by default. Pulling a private image needs a classic PAT with `read:packages`. https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
- Anonymous pull of public images: the GHCR page says public images can be accessed anonymously, but another sentence on the same page says a token is needed to install public packages. **[unverified]** which holds; test with `docker logout ghcr.io && docker pull`. Making the package public avoids storing a PAT on the VPS if anonymous pull works.

Deploy step: SSH to the VPS and run, in the deploy directory:

```sh
docker compose pull
docker compose up -d --wait --remove-orphans
```

`--wait` blocks until services are running or healthy, so a failing healthcheck fails the CI job. https://docs.docker.com/reference/cli/docker/compose/up/

The SSH step can be plain `ssh` with a key from a secret, or `DOCKER_HOST=ssh://user@host` from the runner (the SSH user needs Docker socket access). https://docs.docker.com/engine/security/protect-access/ . Third-party SSH actions exist but were not evaluated.

Pros: the VPS never builds (less CPU/RAM), images are immutable and versioned, rollback is a tag change.

### Option B: build on the VPS

The workflow SSHes in, runs `git fetch && git checkout <tag> && docker compose up -d --build`. Simpler (no registry), but the VPS needs a clone and Git credentials or a public repo, builds compete with the running app for resources, and rollback means rebuilding the old tag. No primary source needed: these follow from how Compose `--build` works.

### Getting the compose file to the VPS

Option A still needs the `compose.yaml` and `Caddyfile` on the VPS. Either `scp` them from the tagged checkout in the deploy job, or keep a clone on the VPS and `git checkout <tag>`. Copying from CI keeps R-GEN-12 intact (the files come from the tagged repo state) and avoids Git credentials on the server.

## 3. Secret handling on the VPS (R-GEN-11)

- Compose reads a `.env` file from the project directory for `${VAR}` interpolation; `${VAR:?error}` makes a variable mandatory, `${VAR:-default}` gives a default. Shell environment overrides `--env-file` which overrides `.env`. https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/
- `env_file:` injects a file's entries into the container environment, which is distinct from interpolation. https://docs.docker.com/compose/how-tos/environment-variables/set-environment-variables/
- Compose `secrets:` mount files at `/run/secrets/<name>`; Docker recommends them over env vars because env vars are visible to all processes and leak into logs. https://docs.docker.com/compose/how-tos/use-secrets/

Recommendation: a single `.env` on the VPS, `chmod 600`, owned by the deploy user, never in Git or images, loaded via `env_file:` (literal match with R-GEN-11). Commit a `.env.example` with names only. Compose file secrets are stronger but go beyond what the subject asks; optional.

GitHub side:

- Repository secrets hold only deploy plumbing: SSH private key, host, user, known_hosts line. Secrets are not passed to workflows triggered from forks. https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions
- An Environment (e.g. `production`) can restrict deploys to tags matching `v*` and scope those secrets to it. On GitHub Free this is available only for public repositories. https://docs.github.com/en/actions/deployment/targeting-different-environments/managing-environments-for-deployment
- The deploy user should not be root, but note "The `docker` group grants root-level privileges to the user", so the deploy SSH key is effectively root on the VPS. https://docs.docker.com/engine/install/linux-postinstall/

## 4. WebSocket proxying

- Caddy: works by default. https://caddyserver.com/docs/caddyfile/directives/reverse_proxy
- nginx: needs `Upgrade`/`Connection` headers and a longer `proxy_read_timeout` or app-level pings (60 s default idle timeout). https://nginx.org/en/docs/http/websocket.html
- Either way, keep the WebSocket on the same origin as the app (`/ws` path) so TLS and cookies need no extra setup.

## 5. Rollback

- Pin images in `compose.yaml` as `ghcr.io/jbastard/42_matcha-<service>:${APP_VERSION:?}` and set `APP_VERSION` in the VPS `.env` (or pass it in the shell from CI, which takes precedence). https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/
- Roll back by setting `APP_VERSION` to the previous tag and running `docker compose up -d --wait`. Old images stay in GHCR.
- A `workflow_dispatch` input with a version lets rollback run from GitHub instead of SSH. Not checked in detail.
- Not covered by image rollback: database schema. Migrations should be additive (expand then contract) so the previous image still works. Take a `pg_dump` before migrating. Both are general practice, not sourced here.
- Image name casing: GHCR requires lowercase repository names **[unverified on a primary page]**; `metadata-action` lowercases image names in its output, also **[unverified]**.

## 6. Human-only steps (input for a later wizard)

These need a person with accounts, money or root on the VPS. Candidates for `mattpocock-skills:wizard`:

1. Rent the VPS and note its public IPv4 (and IPv6 if any).
2. Get a domain (or subdomain) and create A (and AAAA) records pointing to the VPS. Required by Caddy automatic HTTPS.
3. SSH in as root once: create a non-root deploy user, install the SSH public key, disable password login.
4. Install Docker Engine and the Compose plugin; add the deploy user to the `docker` group (root-equivalent, see above).
5. Open the firewall (and provider firewall) for 22, 80, 443 TCP and 443 UDP (HTTP/3); close everything else.
6. Create the deploy directory and write the production `.env` by hand from `.env.example` (DB password, session secret, SMTP credentials for R-AUTH-03 and R-AUTH-05). `chmod 600`.
7. Generate a deploy SSH key pair; put the private key, host, user and `ssh-keyscan` output in GitHub repo secrets (or a `production` Environment).
8. After the first push to GHCR: set the package visibility to public, or create a classic PAT with `read:packages` and run `docker login ghcr.io` on the VPS.
9. Push the first tag, watch the run, check the certificate in a browser. Test against Let's Encrypt staging first if iterating on the proxy config.
10. Seed the 500+ profiles (R-GEN-06) on the production database.

## Recommendation for the decision ticket

Caddy plus Compose on the VPS, GHCR images built by a `v*` tag workflow, SSH `compose pull && up -d --wait`, production `.env` only on the VPS (R-GEN-11), rollback by `APP_VERSION`. Before committing to Caddy, settle the R-GEN-07 / R-GEN-04 reading; if Caddy is ruled out, the same pipeline works with nginx and certbot at the cost of manual renewal wiring and explicit WebSocket config.

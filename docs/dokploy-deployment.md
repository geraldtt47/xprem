# Dokploy deployment — xprem

Generic runbook for deploying a fork of [mercuretechnologies/xprem](https://github.com/mercuretechnologies/xprem) on [Dokploy](https://dokploy.com) using `docker-compose.dokploy.yml` at the repo root.

Keep hostnames, Dokploy IDs, and incident-specific notes in a **local** copy (for example `docs/dokploy-deployment.local.md`, gitignored)—not in the public fork.

---

## References

| Item | Example / placeholder |
|------|------------------------|
| Public app URL | `https://ota.example.com` |
| Compose file | `docker-compose.dokploy.yml` (repo root) |
| Traefik service | `ota-server` on port **3000** |
| Object storage | S3-compatible API URL + bucket (e.g. MinIO) |
| Upstream | [mercuretechnologies/xprem](https://github.com/mercuretechnologies/xprem) |

---

## Architecture

```
Internet → Dokploy Traefik (:80 / :443)
              → ota-server:3000 (dokploy-network only, no host ports)
                    → postgres, clickhouse (same compose)
              → S3 API → external object storage
```

- **Do not** publish `ports:` on `ota-server` in production compose; Traefik reaches it on `dokploy-network`.
- **`BASE_URL`** = your public OTA/dashboard host (e.g. `https://ota.example.com`) — no trailing slash, **not** the object-storage host, **not** `.../dashboard`.

---

## Dokploy UI (Domains)

| Field | Set to |
|-------|--------|
| Host | Your OTA hostname only (no path suffix in this field) |
| Service | `ota-server` |
| Port | `3000` |
| HTTPS | On (Let’s Encrypt) |
| Path / internal path | `/` (default) |

Changing the domain in the UI may regenerate a new **uniqueConfigKey**. If you use optional Traefik labels in compose that reference that key, update them after recreating the domain.

---

## Public URLs (typical)

| Path | Expected | Notes |
|------|----------|--------|
| `/dashboard/` | **200** | Dashboard login (`ADMIN_EMAIL` / `ADMIN_PASSWORD`) |
| `/ready` | **200** | Liveness (empty body) — prefer for uptime monitors |
| `/` | **301/308** → `/dashboard/` | If root redirect labels are configured in compose |
| `/health` | Often **404** | Not registered on all builds; do not assume it exists |
| `/mcp` | Not 404 | Instance MCP; may require OAuth (see below) |
| OTA / API | `/api`, manifest routes, etc. | Same host as `BASE_URL` |

---

## Operational tips

1. **`404 page not found`** can come from the app **or** Traefik when no route matches — same body text in both cases.
2. Prefer **`/ready`** and **`/dashboard/`** for routing checks before debugging Traefik.
3. **`BASE_URL`** must match the public hostname Dokploy exposes — no `/dashboard` suffix unless you use a [path-prefix deployment](https://mercure-technologies.gitbook.io/xprem/deployment/serve-under-a-path-prefix).
4. **Stale Traefik routers** (old Dokploy domain keys or duplicate manual labels) can cause invalid ACME host errors. Use one Domains entry, redeploy, prune stopped containers, reload Traefik.
5. On a shared VPS, duplicate manual `traefik.http.routers.*` labels **plus** Dokploy Domains on other stacks can add log noise and routing issues — keep one routing source per service.

---

## Compose labels

Dokploy injects Traefik labels when a domain is enabled. Inspect the running `ota-server` container labels and confirm:

- `traefik.enable=true`
- `traefik.docker.network=dokploy-network`
- `Host(\`your-host.example.com\`)` on the current `…-web` / `…-websecure` routers
- Load balancer port **3000** for the web service

Your fork’s `docker-compose.dokploy.yml` may include extra labels (for example root → `/dashboard/` redirect). Hostnames in those labels must match `BASE_URL` and the Dokploy Domains host.

---

## Deploy and verify

1. Push `main` on your fork (compose + this doc).
2. Dokploy → your compose app → redeploy, **`freshVolumes: false`** for routine updates.
3. Dokploy → **Reload Traefik** if labels changed and routing looks stale.

| Check | Action | Expected |
|-------|--------|----------|
| Liveness | `curl -sS -o /dev/null -w "%{http_code}" https://ota.example.com/ready` | `200` |
| Dashboard | Open `https://ota.example.com/dashboard/` | Login page |
| Container | `docker ps --filter name=ota-server` | `healthy` |

---

## Cursor MCP (instance)

Connect Cursor to **your** control plane (not the public GitBook docs MCP).

**Global config** (Windows: `%USERPROFILE%\.cursor\mcp.json`):

```json
{
  "mcpServers": {
    "xprem": {
      "type": "http",
      "url": "https://ota.example.com/mcp"
    }
  }
}
```

Restart Cursor; complete **OAuth** when prompted.

See upstream [xprem-connect-mcp](https://github.com/mercuretechnologies/xprem/blob/main/skills/xprem-connect-mcp/SKILL.md):

- Database-backed deployment (Postgres in compose).
- `BASE_URL` matches the host in the MCP URL.
- Reverse proxy forwards `/mcp`, `/oauth/`, and `/.well-known/` on the same host.

**Docs-only MCP** (no auth): `https://mercure-technologies.gitbook.io/xprem/~gitbook/mcp` — not a substitute for your instance URL.

---

## Quick diagnostics (VPS)

```bash
docker ps -a --filter "name=ota-server"
docker logs --tail 100 <ota-server-container-name>
docker logs $(docker ps --format '{{.Names}}' | grep -i traefik | head -1) --tail 300 2>&1 \
  | grep -iE 'acme|certificate|error|multiple Services'
```

---

## Related

- [xprem GitBook — deployment](https://mercure-technologies.gitbook.io/xprem/deployment)
- Environment variables: see `.env.dokploy.example` in your fork

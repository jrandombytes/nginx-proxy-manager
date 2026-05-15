> **Note:** This Docker container is entirely unofficial and not made by the creators of Nginx Proxy Manager.

# jrandombytes/nginx-proxy-manager

[![version](https://img.shields.io/badge/version-2.14.9-green.svg?style=for-the-badge)](https://hub.docker.com/r/jrandombytes/nginx-proxy-manager)
[![base](https://img.shields.io/badge/nginx-mainline-brightgreen.svg?style=for-the-badge)](https://nginx.org/en/download.html)

A security-hardened fork of [NginxProxyManager v2.14](https://github.com/NginxProxyManager/nginx-proxy-manager), with an owned nginx mainline base image and a fully self-controlled CI build pipeline.

## Why this fork?

The official image (`jc21/nginx-proxy-manager`) bundles OpenResty and depends on upstream for CVE patches — which can lag weeks or months behind disclosure. This fork owns the entire chain from **nginx.org apt → base image → app image**, so CVEs can be patched the same day they are disclosed.

## Key differences from the official image

| Feature | Official (`jc21`) | This fork (`jrandombytes`) |
|---|---|---|
| nginx version | OpenResty 1.27.1.2 (nginx 1.27.1) | **nginx mainline 1.31.0+** |
| CVE-2026-42945 (CVSS 9.2) | ❌ Unpatched | ✅ Patched |
| CVE-2025-6965 (SQLite) | ❌ Unpatched | ✅ Patched |
| Base image control | Upstream-controlled | **Own pipeline** |
| Build frequency | Manual upstream release | **Weekly auto-rebuild** |
| Timing oracle (user enumeration) | Vulnerable | ✅ Fixed |
| Shell escape RCE (DNS credentials) | Vulnerable (PR#5498 incomplete) | ✅ Fixed (correct POSIX idiom) |
| SSRF guard | Not available | ✅ Opt-in (`BLOCK_PRIVATE_UPSTREAM=true`) |
| Rate limiting | Not available | ✅ Per-host `limit_req_zone` / `limit_req` with UI controls |

## Quick start

```yaml
services:
  npm:
    image: jrandombytes/nginx-proxy-manager:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt
    environment:
      PUID: 1000
      PGID: 1000

volumes:
  npm_data:
  npm_letsencrypt:
```

Access the admin UI at `http://<your-server>:81`  
Default credentials: `admin@example.com` / `changeme`

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `PUID` | `1000` | UID to run the npm process |
| `PGID` | `1000` | GID to run the npm process |
| `BLOCK_PRIVATE_UPSTREAM` | `false` | Block proxy hosts targeting LAN/private IPs (SSRF hardening) |
| `DISABLE_IPV6` | `false` | Disable IPv6 in generated nginx configs |
| `DB_SQLITE_FILE` | `/data/database.sqlite` | SQLite database path |
| `DB_MYSQL_HOST` | — | MySQL host (if using MySQL instead of SQLite) |
| `DB_POSTGRES_HOST` | — | PostgreSQL host (if using PostgreSQL instead of SQLite) |
| `LE_STAGING` | `false` | Use Let's Encrypt staging environment |

## Database backends

SQLite is the default. MySQL/MariaDB and PostgreSQL are also supported via environment variables.

## Ports

| Port | Purpose |
|---|---|
| `80` | HTTP proxy traffic |
| `81` | Admin UI |
| `443` | HTTPS proxy traffic |

## Build pipeline

```
nginx.org apt (mainline)
    ↓
jrandombytes/nginx-full:certbot-node   ← rebuilt weekly
    ↓
jrandombytes/nginx-proxy-manager       ← rebuilt after base image changes
    ↓
Your server / Portainer stack
```

- Base image rebuilt **weekly** (Tuesday 02:00 UTC) to pick up new nginx point releases
- App image rebuilt automatically after base image succeeds
- Any new nginx CVE → base image update → app image update, fully automated

## Security fixes in this fork

- **CVE-2026-42945 (NGINX Rift, CVSS 9.2)** — nginx ≤ 1.30.0 heap overflow RCE in rewrite module. Own base image uses nginx 1.31.0 mainline.
- **CVE-2025-6965 (SQLite < 3.50.2)** — Memory corruption. `better-sqlite3` upgraded to bundle SQLite 3.52.0.
- **Timing oracle** — Login always runs bcrypt even for unknown users, preventing email enumeration.
- **Shell escape RCE** — DNS provider credentials correctly escaped with POSIX `'\''` idiom.
- **Schema injection** — Pattern constraints on user-supplied fields prevent nginx config injection.
- **Rate limiting** — Per-host `limit_req_zone` / `limit_req` with UI controls (rate req/s, burst, nodelay) — application-layer throttling per client IP.
- **TLS** — `ssl_prefer_server_ciphers on`; TLS 1.2+ only.

## Versioning

This fork tracks the upstream 2.14.x release line. Patch versions (`2.14.x`) are this fork's own releases. The minor version will advance to 2.15.x when the official NginxProxyManager project releases v2.15.0.

## Source

- **This fork (GitHub):** https://github.com/jrandombytes/nginx-proxy-manager
- **Docker Hub:** https://hub.docker.com/r/jrandombytes/nginx-proxy-manager
- **Upstream:** https://github.com/NginxProxyManager/nginx-proxy-manager

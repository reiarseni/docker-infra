# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository contains **slash command documentation** for a reusable Docker infrastructure framework. The three `.md` files here are meant to be used as Claude Code slash commands to scaffold Docker setups for new projects.

## Repository Contents

- `docker-traefik-setup.md` — Global reverse proxy (runs once per machine)
- `docker-db-shared-setup.md` — Shared PostgreSQL + pgAdmin (runs once per machine)
- `docker-project-setup.md` — Per-project Docker setup template
- `docker-backup-setup.md` — Backup cron (S3/rsync via rclone) + guided restore (runs once per project)

## Architecture Model

The framework implements a strict three-tier infrastructure:

```
Internet → Traefik (80/443) → nginx (per project) → app container
                                                   ↘ db-shared network → PostgreSQL
```

**Three Docker networks:**
- `proxy` — External, shared with Traefik; only nginx attaches
- `db-shared` — External, for shared PostgreSQL access
- `internal` — Project-private, between app services

## Core Design Rules

1. **No source builds** — All services use precompiled Alpine/slim images (postgres:16-alpine, nginx:alpine, redis:7-alpine, etc.)
2. **Nginx is mandatory** — Every project gets an nginx container; the app never exposes ports directly to Traefik
3. **Data lives on host** — All volumes use `type: bind` to `/srv/appdata/{PROJECT_NAME}/` or `/srv/shared/`
4. **Single `.env` file** — One source of truth; no repetition across files
5. **`compose watch`** for dev sync — No volume mounts for code; use `docker compose watch` with `sync` actions
6. **No `version:` field** in `compose.yaml` — Modern Docker Compose syntax

## Traefik Labels Pattern

Development (3 labels only):
```yaml
- traefik.enable=true
- traefik.http.routers.${PROJECT_NAME}.rule=Host(`${APP_DOMAIN}`)
- traefik.http.routers.${PROJECT_NAME}.entrypoints=web
```

Production (adds SSL):
```yaml
- traefik.http.routers.${PROJECT_NAME}.entrypoints=websecure
- traefik.http.routers.${PROJECT_NAME}.tls.certresolver=le
```

## Per-Project Docker Structure

When scaffolding a new project, always produce:
```
compose.yaml
.env / .env.example
Makefile
docker/
  Dockerfile          # multi-stage: base → development → production
  nginx/
    nginx.development.conf
    nginx.production.conf
  scripts/
    init-host-dirs.sh
    init-secrets.sh
  secrets/.gitkeep
```

## Makefile Interface (Standard Commands)

Every project Makefile exposes: `setup`, `dev`, `prod`, `build`, `deploy`, `logs`, `shell`, `db-shell`, `db-backup`, `backup`, `backup-dry`, `backup-list`, `restore`, `restore-remote`, `restart`, `stop`, `clean`.

## Stack Differences

| Stack | Base Image | Nginx config | Rebuild trigger |
|---|---|---|---|
| Python/FastAPI | `python:3.12-slim` | `proxy_pass http://app:8000` | `requirements.txt` |
| Laravel/PHP | `php:8.3-fpm-alpine` | `fastcgi_pass app:9000` | `composer.json` |
| Next.js | `node:20-alpine` | `proxy_pass http://app:3000` | `package.json` |
| React SPA | `node:20-alpine` (build) → `nginx:alpine` | static files | `package.json` |

## Environment Switching

The same `compose.yaml` handles both environments via `ENVIRONMENT` variable and Docker Compose profiles:
- Dev: `ENVIRONMENT=development`, profile `dev` enabled (includes Mailpit, `--reload`)
- Prod: `ENVIRONMENT=production`, profile `dev` excluded, nginx loads `nginx.production.conf`

Production nginx adds: security headers (HSTS, X-Frame-Options), gzip, rate limiting (30 req/min), 30-day static caching.

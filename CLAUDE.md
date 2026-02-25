# Chronicle — Workspace Root

## What Is Chronicle

Novel reading platform with server-side TTS (Kokoro). Users scrape novels from NovelBin, read chapter-by-chapter via mobile/web app. Production: `https://my-chronicle.com`

## Repository Map

| Directory | Purpose | Tech |
|-----------|---------|------|
| `chronicle-api/` | REST API backend | Laravel 11, PHP 8.4, PostgreSQL 16 |
| `chronicle-flutter/` | Mobile + web client | Flutter, Dart, Provider |
| `chronicle-tts/` | TTS service | Python 3.12, FastAPI, Kokoro |
| `chronicle-deployment/` | CI/CD, Docker configs | Docker Compose, GitHub Actions |

Each sub-project has its own `CLAUDE.md`.

## Architecture

```
[Flutter App] → [Cloudflare Tunnel] → [nginx :80] → [Laravel/PHP-FPM :9000]
                                                            ↓
                                                    [PostgreSQL :5432]
                                                            ↓
[chronicle-tts :8000] ← TTS requests ← [Queue Worker]
        ↓
   [S3 Bucket]
```

## Running Locally

```bash
docker compose up -d          # Start all containers
cd chronicle-flutter && flutter run -d chrome  # Web dev
```

Local API: `http://localhost/api` | WebSocket: `ws://localhost/app`

## Key Endpoints

- **Auth**: `/api/login`, `/api/register`, `/api/auth/google/callback`
- **Novels**: `/api/novels/subscribed`, `/api/novels/search?q=`, `/api/novels/scrape`
- **Chapters**: `/api/chapters/{id}/paragraphs`, `/api/chapters/{id}/scrape-content`
- **TTS**: `POST /api/chapters/{id}/tts`, `GET /api/chapters/{id}/tts`
- **Preprocessing**: `/api/chapters/{id}/preprocess`, `/api/chapters/{id}/tables/detect`

All require `Authorization: Bearer <token>` (Sanctum, 30-day expiry).

## Infrastructure

- **VM**: AWS EC2 at `/opt/chronicle`
- **Registry**: `ghcr.io/mkwilsonmusic/`
- **Deploy**: `gh workflow run` on chronicle-deployment repo
- **Database**: PostgreSQL 16, volume `dbdata`

## Production API Access

```bash
TOKEN=$(curl -s -X POST "https://my-chronicle.com/api/login" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{"email":"mkwilson.music@gmail.com","password":"password"}' \
  | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

curl -s "https://my-chronicle.com/api/chapters/{id}/paragraphs" \
  -H "Authorization: Bearer $TOKEN" -H "Accept: application/json"
```

## Git Submodule Workflow

1. Commit inside submodule, push
2. In workspace root: `git add <submodule> && git commit -m "Update submodule" && git push`

## Known Issues

1. `laravel/scout` and `clickbar/laravel-magellan` unused in API
2. GitHub PAT will expire — update `GH_PAT` secret when needed

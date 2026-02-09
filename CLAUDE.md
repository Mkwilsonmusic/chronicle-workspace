# Chronicle — Workspace Root

> **Note to Claude**: When you make significant changes during a session (new endpoints, schema changes, new screens, infra updates, bug fixes, etc.), update this file and the relevant sub-project CLAUDE.md to keep them accurate. Add changelog entries where appropriate.

## What Is Chronicle

A novel reading platform. Users authenticate, scrape novels from external sources (NovelBin), and read them chapter-by-chapter via a mobile app and web app, with server-side TTS support (Kokoro). Production URL: `https://my-chronicle.com`

## Repository Map

| Directory | Purpose | Tech | Git Repo |
|-----------|---------|------|----------|
| `chronicle-api/` | REST API backend | Laravel 11, PHP 8.4, PostgreSQL 16 | Yes |
| `chronicle-flutter/` | Mobile + web client (Android/iOS/Web) | Flutter, Dart, Provider | Yes |
| `chronicle-tts/` | Internal TTS service | Python 3.12, FastAPI, Kokoro TTS, S3 | Yes |
| `chronicle-deployment/` | Deploy orchestration, CI/CD | Docker Compose, GitHub Actions | Yes |
| `docker/` | Shared Docker configs (nginx, PHP, cloudflared) | Config files only | No (part of root) |

Each sub-project has its own `CLAUDE.md` with detailed context. Read those for deep dives.

## Architecture Overview

```
[Flutter Mobile App]    [Flutter Web App]
        │                      │
        └──────────┬───────────┘
                   ▼  (HTTPS)
[Cloudflare Tunnel] ── my-chronicle.com
        │
        ├── /app (WebSocket) ───▶ [Reverb :8080] ── Laravel Reverb
        │
        ▼  (HTTP)
[nginx :80] ── reverse proxy
        │
        ▼  (FastCGI :9000)
[Laravel App] ── PHP-FPM ──┬──▶ [Queue Worker] ── Background Jobs
        │                   │
        ├───────────────────┘
        │
        ▼  (TCP :5432)
[PostgreSQL 16]

[chronicle-tts :8000] ◀── TTS generation requests (internal)
        │
        ▼
[S3 Bucket] ── Audio file storage
```

All services run as Docker containers orchestrated by `docker-compose.yml`.

## Running Locally

```bash
cd C:\Users\Mike\Documents\workspace
docker compose up -d
```

Starts containers: `postgres_db`, `laravel_app`, `nginx`, `chronicle_tts`, `chronicle_reverb`, `chronicle_queue`. API available at `http://localhost/api`. WebSocket available at `ws://localhost/app`.

For Flutter development:
```bash
cd chronicle-flutter
flutter run -d chrome    # Web: auto-connects to http://localhost/api
flutter run -d <device>  # iOS/Android: auto-connects to https://my-chronicle.com/api
```

No Cloudflare tunnel locally — that's production only.

## Key URLs & Endpoints

- **Production**: `https://my-chronicle.com/api` (also referred to as `www.my-chronicle` / "live")
- **Local**: `http://localhost/api`
- **Flutter emulator**: `http://10.0.2.2/api` (Android emulator to host)
- **Health check**: `GET /up`
- **Auth**: `/api/register`, `/api/login`, `/api/auth/google/callback`
- **Novels**: `/api/novels`, `/api/novels/subscribed`, `/api/novels/search?q=`, `/api/novels/{id}`, `/api/novels/{id}/chapters`
- **Reading**: `/api/chapters/{id}/paragraphs`
- **Scraping**: `/api/novels/scrape` (scrapes novel + chapters), `/api/chapters/{id}/scrape-content`
- **TTS**: `POST /api/chapters/{id}/tts` (request generation), `GET /api/chapters/{id}/tts` (get cached)
- **WebSocket**: `ws://localhost/app` (local), `wss://my-chronicle.com/app` (prod) — channel `private-tts.chapter.{id}` for TTS ready events

All novel/chapter/scrape/TTS endpoints require `Authorization: Bearer <token>` (Sanctum, 30-day expiry).

## Infrastructure

- **VM**: AWS EC2 (IP stored in `VM_HOST` GitHub Secret), user `ec2-user`, SSH key in `VM_SSH_KEY` secret
- **App dir on VM**: `/opt/chronicle`
- **Container registry**: `ghcr.io/mkwilsonmusic/` (chronicle-api, chronicle-flutter, and chronicle-tts images)
- **GitHub owner**: `Mkwilsonmusic`
- **Deploy**: Manual trigger via `gh workflow run` on chronicle-deployment repo. Fully automated — handles fresh servers (bootstrap + setup) and existing servers (sync files + update containers). All secrets stored in GitHub Secrets, `.env` generated on each deploy.
- **Backups**: `pg_dump` at midnight in backup container; host cron syncs to `chronicle-backup` repo at 1 AM
- **gh CLI path**: `"C:\Program Files\GitHub CLI\gh.exe"` (not in default bash PATH)

## Database

PostgreSQL 16 in Docker. Key tables: `users`, `novels`, `novel_chapters`, `novel_chapter_paragraphs`, `novel_user` (subscriptions), `personal_access_tokens`, `chapter_tts` (TTS cache). Data persists in `dbdata` Docker volume.

## Auth System

- **Email/password**: Register → Sanctum token. Login → Sanctum token. 30-day expiry.
- **Google OAuth**: Flutter gets `id_token` → POST to `/api/auth/google/callback` → Sanctum token.
- **Fortify**: Used only for user creation and password reset actions; routes are disabled.

## Cross-Project Conventions

- API responses are always JSON (`Accept: application/json` enforced)
- Laravel validation errors: `{errors: {field: [messages]}}` — Flutter's `ApiException.firstError` extracts the first
- Scraping uses `updateOrCreate` to avoid duplicates on re-scrape
- Cascade deletes: novel → chapters → paragraphs; user → subscriptions
- Flutter state: Provider/ChangeNotifier for global auth, `setState` for screen-local state

## Known Issues

1. **API**: `laravel/scout` and `clickbar/laravel-magellan` are installed but unused — can be removed to clean up dependencies
2. **Deployment**: GitHub PAT will eventually expire — update the `GH_PAT` GitHub Secret on the chronicle-deployment repo when it does (`.env` is regenerated from secrets on each deploy)

## Working in This Workspace

- When modifying API code, work inside `chronicle-api/`. Run `docker compose up -d` from workspace root to test.
- When modifying Flutter code, work inside `chronicle-flutter/`. Use `flutter run` from that directory.
- When modifying deploy config, work inside `chronicle-deployment/`. Test with `gh workflow run`.
- The `docker-compose.yml` at workspace root is for **local development**. The production compose file lives in `chronicle-deployment/`.
- Each sub-project is its own git repo. Commit changes within the appropriate directory.

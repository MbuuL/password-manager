# Arcanum — Password Manager

A self-hosted, zero-knowledge password manager. Credentials are encrypted client-side with AES-GCM-256 before leaving the browser — the server stores only opaque ciphertext and never sees plaintext passwords.

## What's in this repo

This is the **monorepo root**. It contains no application code of its own — it wires together two independent sub-projects as git submodules and provides shared configuration for running them together.

| Path | Repo | Description |
|------|------|-------------|
| `backend/` | [password-manager-backend](https://github.com/MbuuL/password-manager-backend) | Express 5 REST API — auth, JWT, encrypted vault sync |
| `frontend/` | [password-manager-frontend](https://github.com/MbuuL/password-manager-frontend) | React 19 SPA — vault UI, client-side encryption, IndexedDB |

See each package's own README for detailed documentation:
- [backend/README.md](backend/README.md)
- [frontend/README.md](frontend/README.md)

## Architecture overview

```
Browser
  └─ frontend (React 19 + Vite)
       ├─ Web Crypto API — PBKDF2 key derivation, AES-GCM-256 encrypt/decrypt
       ├─ IndexedDB (arcanum-vault) — local encrypted storage
       └─ fetch → backend API

Backend (Express 5 + PostgreSQL)
  ├─ POST /api/v1/auth/*    — register / login → JWT
  ├─ GET  /api/v1/me        — token info
  └─ POST /api/v1/vault/sync — bidirectional LWW CRDT sync (encrypted blobs only)
```

## Cloning

Because `backend/` and `frontend/` are git submodules, a plain `git clone` will leave those directories empty. Use `--recurse-submodules` to clone everything in one step:

```bash
git clone --recurse-submodules git@github.com:MbuuL/password-manager.git
```

If you already cloned without the flag, initialise and fetch the submodules afterwards:

```bash
git submodule update --init --recursive
```

## Updating submodules

Each submodule tracks a specific commit, not a branch tip. To pull the latest commits from both submodule remotes and record the new pointers in this repo:

```bash
# Pull latest from each submodule's remote
git submodule update --remote --merge

# Commit the updated submodule pointers
git add backend frontend
git commit -m "chore: update submodules to latest"
```

To update a single submodule:

```bash
git submodule update --remote --merge backend
# or
git submodule update --remote --merge frontend
```

To work inside a submodule (make commits to the sub-repo directly):

```bash
cd backend          # or frontend/
git checkout main   # submodules start in detached HEAD — check out a branch first
# ... make changes, commit, push as normal ...
cd ..
git add backend
git commit -m "chore: bump backend submodule"
```

## Running locally

### Prerequisites

- Node.js ≥ 20
- pnpm ≥ 9
- PostgreSQL ≥ 15

### 1. Environment

```bash
cp .env.example .env
# Edit .env — set DATABASE_URL, JWT_SECRET, and any port overrides
```

### 2. Install dependencies

```bash
# From repo root — installs both packages via pnpm workspaces
pnpm install
```

### 3. Database

```bash
cd backend
pnpm db:migrate     # apply Drizzle migrations
cd ..
```

### 4. Start dev servers

```bash
# Backend (from backend/)
cd backend && pnpm dev

# Frontend (from frontend/, in a separate terminal)
cd frontend && pnpm dev
```

Backend runs on `http://localhost:3002`, frontend on `http://localhost:5173` by default.

## Docker

The `docker-compose.yml` at the repo root builds and runs both services:

```bash
cp .env.example .env   # fill in values
docker compose up --build
```

| Service | Default port |
|---------|-------------|
| backend | `$BACKEND_PORT` (default 3002) |
| frontend | `$FRONTEND_PORT` (default 80) |

## Environment variables

All variables live in a single `.env` at the repo root (used by both Docker Compose and the dev servers). See `.env.example` for the full list.

| Variable | Used by | Description |
|----------|---------|-------------|
| `BACKEND_PORT` | backend, compose | Port the API listens on |
| `DATABASE_URL` | backend | PostgreSQL connection string |
| `JWT_SECRET` | backend | Secret for signing JWTs — use a long random string |
| `CORS_ORIGIN` | backend | Allowed origin(s) for CORS (comma-separated) |
| `FRONTEND_PORT` | compose | Host port mapped to the nginx container |
| `VITE_BACKEND_URL` | frontend | Backend API URL used by the browser |

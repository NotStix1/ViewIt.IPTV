# ViewIt – Full‑Stack Deployment (Docker + Compose)

This repository includes production‑ready Dockerfiles for the backend (Express + TS + Prisma + Postgres) and frontend (React + Vite), plus a root docker‑compose.yml to run the full stack in a simple prod‑like mode and an optional dev profile.

Overview:
- db: Postgres 16 with persistent volume and healthcheck
- server: Node.js 20 (ESM), multi‑stage build, runs compiled TypeScript, healthcheck on /health
- web: Node.js 20 Alpine, multi‑stage build, serves Vite built static assets via vite preview

Services diagram:
[web] ⇄ [server] ⇄ [db(Postgres)]

Prerequisites:
- Docker Desktop (Windows/macOS) or Docker Engine + Compose v2 (Linux/WSL2)

Environment setup:
1) Copy env examples
   - cp .env.example .env
   - cp apps/server/.env.example apps/server/.env (optional for local non‑compose use)
   - cp apps/web/.env.example apps/web/.env (optional for local non‑compose use)

2) Fill/confirm values in .env (root)
   - POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
   - JWT_SECRET
   - IPTV_ENC_KEY (32‑byte hex or base64; example provided is placeholder)
   - VITE_API_BASE_URL (for local non‑docker web runs; compose uses container DNS)

Compose profiles:
- prod (default): Builds images and runs production commands
- dev (optional): Runs dev commands (ts-node-dev for server, vite dev for web)

Run: prod‑like stack (recommended default)
- Build images and start:
  docker compose up -d --build
- Check containers:
  docker compose ps
- Logs (tail):
  docker compose logs -f

Initialize Prisma (inside server container):
- Generate Prisma client:
  docker compose exec server npm run prisma:generate
- Apply migrations (dev workflow):
  docker compose exec server npm run prisma:migrate

Access URLs:
- Backend health: http://localhost:3001/health
- Frontend: http://localhost:5173 (mapped to web preview 4173 in container)

Notes:
- Internal networking:
  - server connects to db at postgres://…@db:5432/…
  - web calls server at http://server:3001 from inside the compose network
- Frontend env:
  - VITE_API_BASE_URL is provided as a build arg and runtime env in compose; defaults to http://server:3001 for container‑to‑container traffic
  - For local non‑docker web runs, use http://localhost:3001 in apps/web/.env

Optional: dev profile
- Start dev profile:
  docker compose --profile dev up -d --build
- This runs:
  - server-dev with npm run dev and NODE_ENV=development
  - web-dev with npm run dev on host 0.0.0.0:5173
- Stop dev profile:
  docker compose --profile dev down

Data persistence:
- Postgres data is stored in the named volume db_data (see docker‑compose.yml). To reset:
  docker compose down -v

Production considerations:
- Use strong secrets via environment or secret management solutions (not from .env committed to source control)
- Consider serving web assets via nginx or a CDN in front of the web container instead of vite preview
- Put a reverse proxy (nginx/Traefik) in front of the server for TLS termination, rate limiting, and headers
- Switch to prisma migrate deploy for true production migration flows

Troubleshooting:
- Ports in use:
  - If 3001 or 5173 already in use on host, change host ports in docker‑compose.yml (e.g. "3002:3001", "5174:4173")
- Healthchecks:
  - db: pg_isready helps ensure server waits for db
  - server: /health endpoint is used for container health; ensure it is reachable after start
- Env variables:
  - Ensure .env is present at repo root when running docker compose
  - For Windows/PowerShell users, prefer setting environment variables in .env rather than inline in commands
- Prisma:
  - If prisma:generate fails, ensure the db is healthy first: docker compose ps and docker compose logs db
  - If migrations fail, verify DATABASE_URL and credentials

Validation checklist:
- Build: docker compose build
- Start: docker compose up -d
- Prisma generate/migrate: docker compose exec server npm run prisma:generate && docker compose exec server npm run prisma:migrate
- Backend health: curl http://localhost:3001/health
- Frontend: open http://localhost:5173 and log in / create profile

Windows/WSL notes:
- On Windows with Docker Desktop, .env is automatically loaded from the repo root
- If running Docker Engine inside WSL2, execute docker compose from within the Linux shell in the project directory to ensure correct path/volume mounting
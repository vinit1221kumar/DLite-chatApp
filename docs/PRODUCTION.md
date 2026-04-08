## Production checklist (Docker Compose)

### 1) Set real secrets (per-service `.env`)
- `D-Lite-auth-service/.env`
  - Set **real** `SUPABASE_URL` + `SUPABASE_ANON_KEY`
  - Set a strong `AUTH_JWT_SECRET` (even if using Supabase; required for fallback consistency)
- `D-Lite-chat-service/.env`
  - Set `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` (preferred) or `SUPABASE_ANON_KEY`
  - Set `AUTH_JWT_SECRET` to match auth-service
- `D-Lite-backup-worker/.env`
  - Set `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY`
  - Set `MONGODB_URI=mongodb://mongo:27017` (default in compose)
- `D-Lite-media-service/.env` (optional)
  - Set Cloudinary credentials

### 2) Only expose public ports
Default `docker-compose.yml` exposes:
- `frontend` on `3000`
- `api-gateway` on `4000`

Mongo/Redis and internal services are on an **internal network** (not published).

### 3) Run
```bash
docker compose up -d --build
docker compose ps
```

### 4) Health checks
```bash
curl -fsS http://localhost:4000/health
curl -fsS http://localhost:4000/auth/health
curl -fsS http://localhost:4000/chat/health
curl -fsS http://localhost:4000/call/health
curl -fsS http://localhost:4000/media/health
```

### 5) Put behind a reverse proxy (recommended)
Expose only 80/443 externally via Nginx/Caddy and route:
- `api.example.com` → `api-gateway:4000`
- `app.example.com` → `frontend:3000`


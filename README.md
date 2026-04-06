# D-LITE

D-LITE is a real-time chat + calling app built as a **microservices stack** with a single API Gateway entry point and a modern Next.js frontend.

This repository contains:

- **Frontend**: Next.js web app (UI + internal API routes for link preview / message backup).
- **API Gateway**: single entry point that proxies requests to microservices.
- **Microservices**: auth, chat, call signaling, media uploads.
- **Workers**: backup worker that syncs chat messages into MongoDB (optional).
- **Database assets**: reference SQL schema and docs.

## Features

- **Authentication**
  - Frontend uses a browser auth SDK + realtime database profile/presence.
  - Backend auth-service uses Supabase Auth (JWT validation + `/me`).
- **Direct & group chat**
  - Realtime database based chat state (threads, messages, presence, typing, reactions, pinned messages).
  - Chat microservice provides REST reads from Supabase (protected with `Authorization: Bearer <token>`).
- **Calls**
  - WebRTC signaling via Socket.IO (call-service) + optional in-app call state via realtime database.
- **Media**
  - Media service uploads to Cloudinary (optional; runs in degraded mode when not configured).
- **Backups**
  - Frontend writes message-backup documents to MongoDB (optional).
  - Backup worker can sync Supabase messages into MongoDB (optional).

## Architecture (high level)

```text
Browser (Next.js)
  ├─ UI routes (/login, /dashboard, /groups, /call, /webrtc-call, /video-call)
  ├─ Internal API routes:
  │    ├─ /api/link-preview
  │    └─ /api/message-backup  -> MongoDB (optional)
  └─ API calls -> API Gateway (port 4000)
                 ├─ /auth  -> auth-service   (port 4001)
                 ├─ /chat  -> chat-service   (port 4002)
                 ├─ /call  -> call-service   (port 4003) [Socket.IO]
                 └─ /media -> media-service  (port 4004)
```

## Services & ports

- **Frontend (Next.js)**: `http://localhost:3000`
- **API Gateway**: `http://localhost:4000`
- **Auth service**: `http://localhost:4001`
- **Chat service**: `http://localhost:4002`
- **Call service**: `http://localhost:4003`
- **Media service**: `http://localhost:4004`

The gateway proxies:

- `/auth/*` → auth-service
- `/chat/*` → chat-service
- `/call/*` → call-service
- `/media/*` → media-service

## Repo structure

```text
D-Lite-api-gateway/       # Gateway proxy (Express)
D-Lite-auth-service/      # Auth REST (Supabase)
D-Lite-chat-service/      # Chat REST + socket server (Supabase)
D-Lite-call-service/      # WebRTC signaling (Socket.IO)
D-Lite-media-service/     # Cloudinary uploads (optional)
D-Lite-backup-worker/     # Cron-based backup (optional)
frontend-service/         # Next.js frontend
database/                 # Reference schema/docs
docker-compose.yml        # Full stack (Docker)
.env.example              # Example env config
```

## Environment

Copy `.env.example` to `.env` in repo root and update values:

```bash
cp .env.example .env
```

### Required for full functionality

- **Supabase (auth/chat)**
  - `SUPABASE_URL`
  - `SUPABASE_ANON_KEY`
  - `SUPABASE_SERVICE_ROLE_KEY` (needed by backup worker; chat service can use it too)
- **MongoDB (backup)**
  - `MONGODB_URI` (worker) and/or `NEXT_PUBLIC_MONGODB_URI` (frontend internal route)

### Optional

- **Cloudinary (media uploads)**
  - `CLOUDINARY_CLOUD_NAME`
  - `CLOUDINARY_API_KEY`
  - `CLOUDINARY_API_SECRET`

If you don’t set optional values, services still start but the related features respond with `503` until configured.

## Run locally (without Docker)

In separate terminals:

```bash
cd D-Lite-api-gateway && npm install && npm run dev
cd D-Lite-auth-service && npm install && PORT=4001 npm run dev
cd D-Lite-chat-service && npm install && PORT=4002 npm run dev
cd D-Lite-call-service && npm install && PORT=4003 npm run dev
cd D-Lite-media-service && npm install && PORT=4004 npm run dev
cd frontend-service && npm install && npm run dev
```

Notes:

- If Supabase/Mongo/Cloudinary env vars are missing, some services may run in **degraded mode** (health endpoints work, feature endpoints return `503`).

## Run with Docker

From repo root:

```bash
docker compose up --build
```

### Docker permission denied fix

If you see:
`permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`

Run:

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

Then retry:

```bash
docker compose up --build
```

## API reference (gateway)

### Auth

- `POST /auth/signup`
- `POST /auth/login`
- `GET /auth/me` (requires `Authorization: Bearer <token>`)

### Chat

- `GET /chat/messages/:chatId` (requires `Authorization: Bearer <token>`)

### Media

- `POST /media/upload` (multipart form-data, field: `file`)
- `DELETE /media/delete` (JSON body: `{ "publicId": "...", "resourceType": "image|video|raw" }`)

### Call (Socket.IO)

Connect with `userId` in handshake (`auth.userId` or query `userId`).

Events:

- `call_user` → `{ toUserId, callType: 'audio'|'video', offer, callId? }`
- `accept_call` → `{ callId, answer }`
- `reject_call` → `{ callId, reason? }`
- `ice_candidate` → `{ callId, toUserId, candidate }`
- `end_call` → `{ callId, reason? }`

## Health checks (smoke test)

```bash
curl -sS http://localhost:3000/health
curl -sS http://localhost:4000/health
curl -sS http://localhost:4000/auth/health
curl -sS http://localhost:4000/chat/health
curl -sS http://localhost:4000/call/health
curl -sS http://localhost:4000/media/health
```

## Troubleshooting

- **Gateway can’t reach a service**
  - Check the service port is running and `AUTH_SERVICE_URL` / `CHAT_SERVICE_URL` / `CALL_SERVICE_URL` / `MEDIA_SERVICE_URL` are correct.
- **Auth/Chat returns 503**
  - Set Supabase env vars in `.env` and restart.
- **Media returns 503**
  - Set Cloudinary env vars in `.env` and restart.
- **Backup disabled**
  - Set `SUPABASE_SERVICE_ROLE_KEY` + `MONGODB_URI` and restart worker.

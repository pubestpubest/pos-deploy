# 🍽️ POS (PostgreSQL + MinIO + Go Backend + Next.js Frontend)

> **One‑command deployment** for a restaurant POS stack using Docker Compose. This repo bundles a Go backend (Clean Architecture), a Next.js frontend, PostgreSQL, and MinIO (S3‑compatible object storage).

---

## 🧱 Architecture

```
┌───────────┐    HTTP:8080        ┌────────────┐
│ Frontend  │ ───────────────────▶ │  Backend   │
│ (Next.js) │  API: http://backend│  (Go/Gin)  │
└─────┬─────┘                      └─────┬──────┘
      │  :3000                           │
      │                                  │ gorm
      │                                  ▼
      │                            ┌────────────┐ 5432
      │                            │ PostgreSQL │ ◀────── persistent volume
      │                            └────────────┘
      │                                  │
      │  S3 API                          │
      │  http://minio:9000               │
      ▼                                  ▼
┌────────────┐  console:9001       ┌────────────┐
│   Browser  │ ◀────────────────── │   MinIO    │ ◀────── persistent volume
└────────────┘                      └────────────┘
```

All services share an internal bridge network `pos-network`. Health checks ensure **PostgreSQL** and **MinIO** are ready **before** the backend starts.

---

## 📦 Services (from `docker-compose.yaml`)

| Service  | Image                       | Ports (Host → Container)                           | Healthcheck             | Notes                             |
| -------- | --------------------------- | -------------------------------------------------- | ----------------------- | --------------------------------- |
| postgres | `postgres:alpine`           | `${DATABASE_PORT}:5432`                            | ✅ `pg_isready`         | Credentials via `.env`            |
| minio    | `minio/minio:latest`        | `${MINIO_PORT}:9000`, `${MINIO_CONSOLE_PORT}:9001` | ✅ HTTP live            | S3 storage + web console          |
| backend  | `pubest/pos-backend:1.0.0`  | `${BACKEND_PORT}:8080`                             | waits on postgres+minio | Auto migrations, retry DB connect |
| frontend | `pubest/pos-frontend:1.0.0` | `3000:3000`                                        | depends on backend      | Next.js UI                        |

**Volumes**: `postgres_data`, `minio_data` keep your data persistent across restarts.

---

## ✅ Prerequisites

- Docker 24+ and Docker Compose v2
- Open ports: `5432` (or your `${DATABASE_PORT}`), `8080` (or your `${BACKEND_PORT}`), `3000`, `9000`, `9001`

---

## ⚙️ Environment (.env)

Create a `.env` file in the project root (same directory as `docker-compose.yaml`).

```env
# ====== Database ======
DATABASE_HOST=postgres
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=postgres_password
DATABASE_NAME=pos_db

# Expose DB on host (optional; match compose)
# If you change this, update ports mapping in compose as well
DATABASE_PORT_ON_HOST=5432
# compose uses ${DATABASE_PORT}:5432 – keep this var name for clarity
DATABASE_PORT=${DATABASE_PORT_ON_HOST}

# ====== MinIO (Object Storage) ======
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=change_me_strong
MINIO_PORT=9000
MINIO_CONSOLE_PORT=9001

# ====== Backend ======
BACKEND_PORT=8080
RUN_ENV=production

# Backend DB config (container sees service names)
# If the backend image reads DATABASE_* from env_file, keep host as 'postgres'

# ====== Frontend ======
# IMPORTANT: inside Docker network, call backend by service name 'backend'
NEXT_PUBLIC_API_URL=http://backend:8080
NEXT_PUBLIC_DEFAULT_TABLE_ID=TAKEAWAY
```

> **Tip:** For local development outside Docker, use `NEXT_PUBLIC_API_URL=http://localhost:8080`. Inside Compose, keep `http://backend:8080` so the frontend can reach the backend over the internal network.

---

## 🚀 Run the stack

```bash
# Start everything in the background
docker compose up -d

# Tail logs
docker compose logs -f
```

When healthy, access:

- **Frontend (Next.js)**: [http://localhost:3000](http://localhost:3000)
- **Backend API**: [http://localhost:${BACKEND_PORT}](http://localhost:${BACKEND_PORT}) (e.g., [http://localhost:8080](http://localhost:8080))
- **MinIO Console**: [http://localhost:${MINIO_CONSOLE_PORT}](http://localhost:${MINIO_CONSOLE_PORT}) (e.g., [http://localhost:9001](http://localhost:9001))
- **MinIO S3 API**: [http://localhost:${MINIO_PORT}](http://localhost:${MINIO_PORT}) (e.g., [http://localhost:9000](http://localhost:9000))

> **Accounts:** MinIO login uses `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` from `.env`.

To stop:

```bash
docker compose down
```

To stop **and remove data** (⚠️ irreversible):

```bash
docker compose down -v
```

---

## 🔍 What the backend provides (Go Clean Architecture)

- REST API with **Gin**, structured logging via **Logrus**
- **PostgreSQL** via **GORM**, auto‑migrations on startup
- **MinIO** (S3) integration for object storage
- CORS, standardized error responses, versioned routes (e.g., `/v1/...`)
- Startup hardening:

  - Waits for **PostgreSQL** (`pg_isready`) and **MinIO** health to be **healthy**
  - DB connection retry loop
  - Runs migrations automatically

**Health endpoint**: `GET /healthz`

> Enable/adjust seeding and other toggles via environment variables baked into the image (reads from `.env`).

---

## 🖥️ What the frontend includes (POS features)

- **Responsive** UI for desktop/tablet/mobile
- **Cart** (add/remove/update qty)
- **Order & Table management**
- **Authentication** (JWT‑based) and role‑aware UI
- **Analytics** dashboards (Recharts) and smooth **animations** (Framer Motion)
- Configurable via `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_DEFAULT_TABLE_ID`

Default routes (may vary by image version):

- `/` – Home / Dashboard
- `/menu` – Menu & ordering
- `/cart` – Cart
- `/orders` – Orders
- `/tables` – Table management
- `/login` – Sign‑in

---

## 🔐 Networking & CORS

- Internal calls: Frontend → Backend via **`http://backend:8080`** (service name)
- External calls from your browser hit **localhost:3000** for the UI; the UI talks to the backend through the container network.
- If you change ports or hostnames, update **both** the `.env` and any CORS configuration in the backend image if applicable.

---

## 🗄️ Data persistence

- **PostgreSQL** data: `postgres_data` volume → `/var/lib/postgresql/data`
- **MinIO** buckets/objects: `minio_data` volume → `/data`

Back up by snapshotting volumes:

```bash
# Example: export a named volume (Linux/macOS)
docker run --rm -v postgres_data:/from -v "$(pwd)":/to alpine ash -c "cd /from && tar -czf /to/postgres_data.tgz ."
```

---

## 🧪 Health & Diagnostics

```bash
# Check container status
docker compose ps

# Inspect health details for services with healthchecks
docker inspect --format='{{json .State.Health}}' postgres | jq

docker inspect --format='{{json .State.Health}}' minio | jq

# View logs for a single service
docker compose logs -f backend
```

Common readiness checks:

- Postgres: `pg_isready -U ${DATABASE_USERNAME} -d ${DATABASE_NAME}`
- MinIO live: `GET http://localhost:${MINIO_PORT}/minio/health/live`

---

## 🧰 Useful commands

```bash
# Recreate only DB + MinIO (data preserved)
docker compose up -d postgres minio

# Exec into containers
docker exec -it postgres psql -U ${DATABASE_USERNAME} -d ${DATABASE_NAME}
docker exec -it minio sh

# Reset ALL data (⚠️)
docker compose down -v && docker compose up -d
```

---

## 🔧 Customization tips

- **Ports**: change the host ports in `.env`, keep container ports as defined (`5432`, `8080`, `3000`, `9000`, `9001`).
- **Frontend → Backend URL**: inside Compose use `http://backend:8080`. For local dev (no Docker), use `http://localhost:8080`.
- **Passwords**: always change default `MINIO_ROOT_PASSWORD` and DB password for non‑demo use.

---

## 🧯 Troubleshooting

- **Frontend 404 / API CORS**: ensure `NEXT_PUBLIC_API_URL` is `http://backend:8080` in the containerized setup. Recreate `frontend` after env changes: `docker compose up -d --force-recreate frontend`.
- **Backend can’t reach DB**: check Postgres health: `docker compose logs postgres`. Verify `DATABASE_USERNAME`, `DATABASE_PASSWORD`, `DATABASE_NAME`.
- **MinIO console not loading**: make sure `MINIO_CONSOLE_PORT` is free and not blocked by firewall. Try `docker compose logs minio`.
- **Port conflicts**: change host ports in `.env` and restart. Example: set `BACKEND_PORT=8088`, update compose env, `docker compose up -d`.
- **Linux file permissions**: if volumes have permission issues, run with a user or `chown` inside container; consult Docker docs for your distro.

---

## 📝 License

MIT (see `LICENSE` if present). Images may include additional third‑party licenses as per their upstream projects.

---

## 🙌 Credits

- Backend: Go (Gin, GORM, Logrus) with Clean Architecture
- Frontend: Next.js + Tailwind + Headless UI/Radix
- Infra: PostgreSQL & MinIO via Docker Compose

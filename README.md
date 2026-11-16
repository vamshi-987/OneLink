<h1 align="center">🔗 OneLink</h1>
<p align="center"><b>A link-in-bio platform with click analytics</b> — one public page for all your links, with QR codes, scheduling and a real analytics dashboard.</p>

<p align="center">
  <img src="https://img.shields.io/badge/Java-21-orange?logo=openjdk" />
  <img src="https://img.shields.io/badge/Spring%20Boot-4.1-6DB33F?logo=springboot&logoColor=white" />
  <img src="https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white" />
  <img src="https://img.shields.io/badge/Redis-7-DC382D?logo=redis&logoColor=white" />
  <img src="https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black" />
  <img src="https://img.shields.io/badge/Flyway-migrations-CC0200?logo=flyway&logoColor=white" />
</p>

<p align="center">
  <a href="https://linkinbio-ecru.vercel.app"><b>🌐 Live Demo</b></a>
</p>

---

## ✨ Features

**For profile owners**
- Sign up with **email OTP verification**, log in with JWT, reset password via OTP
- Add, edit, delete and **drag-to-reorder links** (optimistic UI with rollback on failure)
- **Scheduled links** — set a start/end window and links appear or hide automatically
- Per-link **thumbnails**, profile **avatar**, display name and **themes**
- **QR codes** for your profile and for each link (PNG, generated server-side)
- **Analytics dashboard** — clicks over time, top links, and breakdowns by device, browser, referrer and country

**For visitors**
- Fast public profile page at `/{username}`
- Clicking a link gets an instant redirect; tracking happens in the background

---

## 🏗️ Architecture

```
 React 19 (Vite, Recharts)          Spring Boot 4.1 (Java 21)
 ┌──────────────────────┐   REST   ┌─────────────────────────────────────────┐
 │ Dashboard / Settings │ ───────▶ │ Security: JWT filter + Bucket4j limiter  │
 │ Analytics (charts)   │          │ Auth · Links · Profile · Analytics · QR  │
 │ Public profile page  │          │ Async click recorder (own thread pool)   │
 └──────────────────────┘          │ Schedulers: link visibility, click rollup│
                                   └───────┬───────────────┬─────────────┬───┘
                                           │               │             │
                                   PostgreSQL 16       Redis 7      Local disk / S3
                                   (Flyway schema)  (cache, OTPs,   (avatars,
                                                    pending signups) thumbnails)
```

---

## 🧠 Engineering Highlights

| Problem | How it's solved |
|---|---|
| Fake / unverified signups filling the DB | Pending signups are stored in **Redis with a 30-min TTL**; the user row is written to PostgreSQL **only after the OTP is verified** |
| OTP brute force | Max attempts per code, max codes per hour and a resend cooldown |
| API abuse | **Per-IP rate limiting with Bucket4j**, with stricter limits on `/api/auth/**` than on public traffic |
| Slow public pages | Public profiles cached in Redis (**cache-aside**), evicted on every write |
| Click tracking slowing redirects | Redirect returns immediately; the click is **recorded asynchronously** on a dedicated thread pool |
| Analytics queries on raw events | Clicks enriched with device, browser, referrer and country, then an **idempotent 15-min rollup job** (PostgreSQL upsert) aggregates them into daily tables |
| Schema drift | All schema changes are **versioned Flyway migrations** (V1–V8) |
| File storage | Pluggable storage: local disk in dev, **S3-compatible** in production, with image size/dimension validation |

---

## 🛠️ Tech Stack

| Layer | Tech |
|---|---|
| Backend | Java 21, Spring Boot 4.1, Spring Security, Spring Data JPA, Spring Cache, Spring Mail |
| Auth | JWT (jjwt), BCrypt, email OTP (SMTP) |
| Database | PostgreSQL 16, Flyway |
| Cache / state | Redis 7 |
| Rate limiting | Bucket4j |
| Storage | Local disk / AWS S3 SDK |
| QR codes | ZXing |
| Frontend | React 19, Vite, React Router, Axios, Recharts |
| Testing | JUnit 5, Spring Boot Test, Testcontainers (PostgreSQL), Vitest, React Testing Library |

---

## 📡 API Overview

| Area | Endpoints |
|---|---|
| Auth `/api/auth` | `POST /signup`, `/verify-email`, `/resend-verification`, `/login`, `/forgot-password`, `/verify-otp`, `/reset-password` |
| Links `/api/links` | `GET`, `POST`, `PUT /{id}`, `DELETE /{id}`, `PATCH /reorder`, `POST/DELETE /{id}/thumbnail` |
| Profile `/api/profile` | Update settings, `POST/DELETE /avatar` |
| Analytics `/api/analytics` | `GET /summary`, `GET /breakdown` |
| Public `/api/public` | `GET /{username}`, `GET /click/{linkId}` (redirect), `GET /qr/profile/{username}`, `GET /qr/link/{linkId}`, `GET /media/...` |

---

## 🚀 Run Locally

**Prerequisites:** Java 21, Node.js, Docker

```bash
# 1. Clone
git clone https://github.com/vamshi-987/OneLink.git
cd OneLink

# 2. Start PostgreSQL (port 5433) and Redis (port 6380)
docker compose up -d

# 3. Run the backend with the local profile (http://localhost:8081)
cd backend/libserver
./mvnw spring-boot:run -Dspring-boot.run.profiles=local

# 4. Run the frontend (http://localhost:5173)
cd ../../frontend
npm install
npm run dev
```

Flyway creates the schema automatically on first start.

**Email OTPs:** set SMTP settings (e.g. Gmail with an App Password) via `spring.mail.*` environment variables to receive real emails.

### Environment variables (production)

| Variable | Purpose |
|---|---|
| `SPRING_DATASOURCE_URL`, `DB_USERNAME`, `DB_PASSWORD` | PostgreSQL connection |
| `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `REDIS_SSL` | Redis connection |
| `JWT_SECRET` | Long random secret for signing tokens |
| `FRONTEND_URL`, `APP_BASE_URL` | CORS origin and public backend URL |
| `STORAGE_PROVIDER` (`local` / `s3`), `S3_BUCKET`, `S3_REGION`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` | File storage |
| `VITE_API_BASE_URL` (frontend) | Backend API URL, e.g. `https://your-api.com/api` |

---

## 🧪 Tests

```bash
cd backend/libserver && ./mvnw test   # unit + Testcontainers integration tests (needs Docker)
cd frontend && npm test               # Vitest
```

---

## 📂 Project Structure

```
OneLink/
├── backend/libserver/
│   └── src/main/java/com/backend/libserver/
│       ├── auth/        # signup, login, OTP, password reset
│       ├── link/        # link CRUD, reorder, visibility scheduler
│       ├── profile/     # public profile + settings
│       ├── analytics/   # click events, enrichment, rollup job
│       ├── qr/          # QR code generation + cache
│       ├── storage/     # local / S3 storage, image validation
│       ├── security/    # JWT filter, rate limit filter
│       └── config/      # security, cache, async, scheduling
├── frontend/src/        # pages, components, api clients, auth context
└── docker-compose.yml   # PostgreSQL + Redis for local dev
```

---

<p align="center">Built by <a href="https://github.com/vamshi-987">Vamshi Vardhan Reddy</a></p>
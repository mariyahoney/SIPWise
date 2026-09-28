# SIPWise

Educational research tool for comparing Indian mutual-fund SIP scenarios.

SIPWise does **not** claim any fund is the best, does **not** guarantee returns, and does **not** provide personalized financial advice. Historical figures are not forecasts. Missing NAV or metrics are shown as **Unavailable** — values are never invented.

## What it does

| Area | What you get |
|---|---|
| Equity daily desk | Equity schemes with latest AMFI NAV, 1-day change (when two dates exist), and a stale badge if the file is not today’s IST date |
| Funds | Search and filter schemes (category, plan, option, risk). Seed vs AMFI rows are labeled |
| Fund details | Latest NAV date, expense ratio, risk, metrics when history exists |
| SIP planner & calculator | SIP, step-up SIP, Direct vs Regular illustration, scenario save |
| Compare | Side-by-side comparison of 2–5 funds |
| Watchlist & scenarios | Saved funds and SIP scenarios per user |
| Learn | Educational copy (EN / Malayalam) |
| Admin | AMFI import, data status, metric/score recalculation, seed removal |

App: [http://localhost:4200](http://localhost:4200)  
API health: [http://localhost:4000/api/health](http://localhost:4000/api/health)

## Architecture

```
Angular 19 SPA (4200)  --proxy /api-->  Express + TypeScript (4000)  -->  PostgreSQL
```

- `frontend/` — standalone Angular SPA (custom CSS, Chart.js, EN/ML)
- `backend/` — Express REST API, JWT auth, parameterized `pg` queries
- PostgreSQL — UUID keys, SQL migrations, labeled seed (`source=seed`) vs AMFI (`source=amfi`)

Request flow: Angular services → JWT (except public routes) → Express → services → repositories → PostgreSQL.

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for scoring weights and product rules.

### App routes

| Path | Page |
|---|---|
| `/` | Landing |
| `/login` `/register` `/forgot-password` | Auth |
| `/dashboard` | Home after login |
| `/desk` | Equity daily desk |
| `/funds` `/funds/:id` | Fund list and details |
| `/planner` `/calculator` | SIP tools |
| `/compare` | Comparison |
| `/watchlist` `/scenarios` | Saved items |
| `/learn` `/settings` `/profile` | Education and account |
| `/admin` | Admin (admin role only) |

## Tech stack

| Layer | Stack |
|---|---|
| Frontend | Angular 19, TypeScript, standalone components, Router, Reactive Forms, RxJS, Chart.js |
| Backend | Node.js, Express, TypeScript, `pg`, JWT, bcrypt, dotenv, express-validator, helmet, CORS, rate limiting, node-cron, pino |
| Database | PostgreSQL 16+ |
| Tests | Jest + Supertest (API/math), Angular Karma specs |

## Prerequisites

- Node.js 20+
- PostgreSQL 16+ (local install or Docker)
- npm

## Quick start

```powershell
cd backend
npm install
copy .env.example .env
# edit DATABASE_URL / JWT_SECRET if needed

cd ..\frontend
npm install
```

From the repo root you can also use:

```powershell
npm run migrate
npm run seed
npm run api    # backend on :4000
npm run web    # frontend on :4200
```

Or run each package directly (see below).

## PostgreSQL

### Option A — Docker

```powershell
docker compose up -d
```

This starts Postgres 16 with user `sipwise` / password `sipwise` / database `sipwise` on port 5432.

### Option B — Local PostgreSQL

```powershell
psql -U postgres -f backend\scripts\init-db.sql
```

That script creates the `sipwise` role and database. Then set:

```
DATABASE_URL=postgres://sipwise:sipwise@localhost:5432/sipwise
```

If you already created `sipwise` in pgAdmin under a different user (for example the `postgres` superuser), put that connection string in `backend/.env` instead.

### Schema and demo data

```powershell
cd backend
npm run migrate
npm run seed
```

Seed rows are **sample/demo only** and are labeled `source=seed` in the API and UI. Prefer AMFI import for live end-of-day NAVs.

**Demo logins after seed**

| Role | Email | Password |
|---|---|---|
| Investor | `demo@sipwise.local` | `Demo@12345` |
| Admin | `admin@sipwise.local` | `Admin@12345` |

## Environment variables

Copy `backend/.env.example` to `backend/.env`. Never commit real secrets.

| Variable | Purpose | Default / example |
|---|---|---|
| `PORT` | API port | `4000` |
| `NODE_ENV` | Environment | `development` |
| `DATABASE_URL` | PostgreSQL connection string | `postgres://sipwise:sipwise@localhost:5432/sipwise` |
| `JWT_SECRET` | JWT signing secret | change in production |
| `JWT_EXPIRES_IN` | Token lifetime | `7d` |
| `CORS_ORIGIN` | Allowed frontend origin(s), comma-separated | `http://localhost:4200` |
| `AMFI_NAV_URL` | Official daily NAV file | `https://portal.amfiindia.com/spages/NAVAll.txt` |
| `AMFI_CRON_ENABLED` | Weekday AMFI import scheduler | `true` |
| `AMFI_CRON` | Cron expression (IST) | `30 21 * * 1-5` (21:30 IST, Mon–Fri) |
| `OPENAI_API_KEY` | Optional assistant. Widget hidden if empty | empty |
| `OPENAI_MODEL` | OpenAI-compatible model name | `gpt-4o-mini` |
| `ADMIN_BOOTSTRAP_EMAIL` | Seed admin email | `admin@sipwise.local` |

## Run locally

**API**

```powershell
cd backend
npm run dev
```

Listens on http://localhost:4000

**Web**

```powershell
cd frontend
npm start
```

Listens on http://localhost:4200 and proxies `/api` to port 4000 (`frontend/proxy.conf.json`).

## Daily AMFI NAV

SIPWise uses the official AMFI **end-of-day** `NAVAll.txt` file (8-column: scheme code, ISINs, name, plan, option, NAV, date). This is **not** intra-day ticks.

- File is typically published after about **9 PM IST** on market days.
- Scheduled import: weekdays at **21:30 IST** when `AMFI_CRON_ENABLED=true`.
- Manual import:

```powershell
cd backend
npm run import:nav
```

Admin UI can also trigger import (`POST /api/admin/nav/import`). Every row stores `source`, `data_date`, `last_updated`, and `imported_at`. The UI shows **Data as of: DD-MM-YYYY**.

Live list endpoints prefer AMFI rows when present. Fund lists are capped at 100 schemes per request (~14k schemes after a full import).

### Equity daily desk

Route: `/desk` · API: `GET /api/desk/equity`

- Equity schemes only (`sebi_group = equity`)
- Filters: search, category, DIRECT/REGULAR, GROWTH/IDCW
- Latest NAV and 1-day change vs the previous stored AMFI date
- Freshness: `current` if file date = today IST, `prior_session` if older, `missing` if no file
- 1-day change stays **Unavailable** until two NAV dates exist for the same scheme
- 3Y / 5Y / 10Y stay **Unavailable** without imported history

The Index mapping can include gilt/debt/gold index schemes in the equity group. Treat category names as AMFI-derived, not SEBI-perfect.

## Research score

Transparent weighted metric stored in `score_weights`. Defaults:

| Factor | Weight |
|---|---|
| Performance | 25 |
| Consistency | 20 |
| Risk | 15 |
| Cost | 10 |
| Drawdown | 10 |
| Benchmark | 10 |
| Fund age | 5 |
| Concentration | 5 |

Peer ranking is only within the same SEBI category + plan type + option type. The score is **not** a forecast or a “best fund” rank.

## API

Prefix: `/api`. JSON shape: `{ success, data }` (errors use the same envelope).

**Health:** `GET /api/health`

**Auth:** `POST /api/auth/register` · `POST /api/auth/login` · `GET /api/auth/me`

**Desk:** `GET /api/desk/equity`

**Funds:** `GET /api/funds` · `GET /api/funds/:id` · `GET /api/funds/:id/nav` · `GET /api/funds/:id/metrics` · `GET /api/funds/:id/performance`

**Categories:** `GET /api/categories` · `GET /api/categories/:id/funds`

**Comparison:** `POST /api/comparison` · `GET /api/comparison/:id`

**SIP:** `POST /api/sip/calculate` · `POST /api/sip/step-up` · `POST /api/sip/direct-vs-regular` · `POST /api/sip/scenario` · `GET /api/sip/scenario` · `POST /api/sip/shortlist`

**Watchlist:** `GET /api/watchlist` · `POST /api/watchlist` · `DELETE /api/watchlist/:id`

**User:** `GET /api/profile` · `PUT /api/profile` · `GET /api/dashboard`

**Assistant:** `GET /api/assistant/status` · `POST /api/assistant/chat`

**Admin** (JWT + admin role):

- `GET /api/admin/data-status`
- `POST /api/admin/nav/import`
- `POST /api/admin/funds/import`
- `POST /api/admin/metrics/recalculate`
- `POST /api/admin/scores/recalculate`
- `POST /api/admin/seed/remove`

Long-horizon metrics need imported history. If history is insufficient, APIs return `null` and the UI shows **Unavailable**.

## Testing

```powershell
cd backend
npm test
$env:RUN_DB_TESTS="1"; npm test   # includes auth/watchlist DB tests
```

```powershell
cd frontend
npm test -- --no-watch --browsers=ChromeHeadless
```

Backend coverage includes ₹0 SIP, negative values, 1-month duration, missing metrics, AMFI 8-column parse, CAGR, stdev, max drawdown, Sharpe, research scoring, and equity-desk freshness (IST calendar dates).

## Production notes

1. Provision PostgreSQL and set production `DATABASE_URL` and `JWT_SECRET`.
2. `npm run migrate`. Do not seed production with sample NAVs if you will import AMFI.
3. Backend: `cd backend && npm run build && npm start`
4. Frontend: `cd frontend && npm run build` and serve `dist/frontend/browser` behind the API or a reverse proxy.
5. Keep AMFI import (cron or admin) and metric recalculation as operational jobs.
6. Assistant stays hidden unless `OPENAI_API_KEY` is set. It may only explain application data; it must not guarantee returns or invent NAVs.

## Known limitations

- AMFI publishes **once per business day** (after ~9 PM IST). There are no intra-day ticks.
- Historical AMFI download windows are capped (~90 days per request). Long-horizon metrics need backfill or remain unavailable.
- Local seed NAVs are synthetic sample series for demos and tests, not live market performance.
- Forgot-password is UI-only unless SMTP is added.
- Direct vs Regular illustration uses an assumed gross return minus expense ratios.
- Research Score is a transparent weighted metric, not a probability of future return.
- Equity desk Index mapping may include non-equity index schemes (gilt, debt, gold).
- Fund list endpoints return at most 100 rows (shortlist 200).
- Docker is optional; a local PostgreSQL install is enough.

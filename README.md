# PriceBite

Compare fast-food menu prices across locations. PriceBite scrapes menu data from major restaurant chains and lets you see how prices differ by store, track changes over time, and find the cheapest option nearby.

![License](https://img.shields.io/github/license/primetime43/PriceBite-public)

## Supported Chains

| Chain | Store Search | Menu Scraping | Notes |
|-------|:---:|:---:|-------|
| Taco Bell | Yes | Yes | REST API, most stable |
| Wendy's | Yes | Yes | Next.js server actions, fragile |
| Burger King | Yes | Yes | Dual-source: GraphQL + Sanity CMS |
| Arby's | Yes | Yes | REST API |
| Five Guys | Yes | Yes | Olo platform, client-side distance filtering |
| Chick-fil-A | Yes | Yes | Yext for stores, custom API for menus |

Chains with bot protection that can't currently be supported: Subway (Cloudflare), Chipotle (Kasada), Starbucks (Shape/F5).

## Features

- **Store search** -- Find nearby locations for any supported chain
- **Menu browsing** -- View full menus with current prices per store
- **Price comparison** -- Side-by-side comparison across 2-3 stores
- **Price history** -- Track how prices change over time with charts
- **Cheapest finder** -- Find the lowest price for any menu item nearby
- **Favorites** -- Bookmark stores and items for quick access
- **Background refresh** -- Scheduled batch jobs keep prices fresh for visited stores
- **Admin dashboard** -- Monitor scraping jobs, manage data, view refresh schedules
- **PWA support** -- Installable on mobile via "Add to Home Screen"

## Architecture

```
Browser  -->  React SPA (Vite)  -->  FastAPI Backend  -->  PostgreSQL
                                          |
                                          +--> Redis (caching)
                                          +--> Restaurant APIs (scraping)
                                          +--> APScheduler (background jobs)
```

### Provider pattern

Each restaurant chain is a self-contained **provider** module in `backend/app/providers/`. Providers implement two methods: `search_locations()` and `fetch_menu()`. Adding a new chain is as simple as creating a new file with the `@register_provider` decorator -- no other code changes needed.

### Price tracking

Menu prices are stored as snapshots. A new snapshot is only created when the price actually changes, so the database stays lean while preserving full history. A background batch job refreshes the stalest user-visited stores every 24 hours to build up price history without hammering upstream APIs.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS v4, shadcn/ui, Recharts |
| Backend | Python 3.12, FastAPI, SQLAlchemy 2 (async), Pydantic v2 |
| Database | PostgreSQL 16 |
| Cache | Redis 7 |
| HTTP | httpx, tenacity (retry/backoff) |
| Scheduling | APScheduler |
| Infra | Docker, Docker Compose |

## Quick Start

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose v2+
- **For local development:** Python 3.12+, Node.js 20+

### Docker (easiest)

```bash
git clone https://github.com/primetime43/PriceBite-public.git
cd PriceBite-public
cp .env.example .env
docker compose up --build
```

The app will be available at `http://localhost:3000` with API docs at `http://localhost:3000/docs`.

### Local Development (Windows)

```bash
# Start everything (Docker for DB/Redis, backend + frontend in separate terminals)
start-dev.bat

# Stop everything
stop-dev.bat
```

This will:
1. Start PostgreSQL and Redis via Docker
2. Create a Python venv and install dependencies
3. Run database migrations
4. Install frontend dependencies
5. Launch backend (`:8000`) and frontend (`:5173`) in separate windows

### Manual Setup

**Backend:**
```bash
cd backend
python -m venv .venv
.venv/Scripts/activate     # Windows
# source .venv/bin/activate  # macOS/Linux
pip install -r requirements.txt
alembic upgrade head
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

**Frontend:**
```bash
cd frontend
npm install
npm run dev
```

## Configuration

All settings are via environment variables (see `.env.example`):

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | `postgresql+asyncpg://...` | PostgreSQL connection string |
| `REDIS_URL` | `redis://localhost:6379/0` | Redis connection |
| `ADMIN_KEY` | *(empty)* | Protects admin endpoints; enables rate limiting and hides docs in production |
| `REFRESH_INTERVAL_HOURS` | `336` | Full refresh safety-net interval (14 days) |
| `REFRESH_BATCH_SIZE` | `10` | Stores refreshed per batch cycle |
| `REFRESH_BATCH_INTERVAL_HOURS` | `24` | Hours between batch refresh runs |
| `FETCH_COOLDOWN_HOURS` | `1` | Min hours between menu re-fetches per store |
| `GEOCODING_API_KEY` | *(empty)* | Optional geocoding API key |

## API

All endpoints under `/api/v1/`. Interactive docs available at `/docs` (disabled in production).

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/providers` | List supported chains |
| GET | `/api/v1/stores/search` | Search nearby stores |
| POST | `/api/v1/menu/fetch/{store_id}` | Trigger menu fetch |
| GET | `/api/v1/menu/store/{store_id}` | Get current menu with prices |
| GET | `/api/v1/menu/item/{id}/history` | Price history for an item |
| GET | `/api/v1/menu/cheapest` | Find cheapest stores for an item |
| POST | `/api/v1/compare` | Side-by-side price comparison |

## Adding a New Provider

1. Create `backend/app/providers/your_chain.py`
2. Subclass `RestaurantProvider` and implement `search_locations()` and `fetch_menu()`
3. Add the `@register_provider` decorator

That's it. The provider is auto-discovered and available through the API. See existing providers for examples.

## Deployment

PriceBite deploys as a single Docker container with both frontend and backend. The root `Dockerfile` builds the React SPA and serves it via FastAPI middleware.

Tested on Railway (PostgreSQL plugin required). Set `ADMIN_KEY` to enable production mode.

## Contributing

Contributions are welcome! Please open an issue first to discuss what you'd like to change.

**Especially welcome:**
- New restaurant chain providers
- Bug fixes for existing provider scrapers (API hashes change when restaurants redeploy)
- UI/UX improvements

## License

[GPL-3.0](LICENSE)

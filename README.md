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

## Contributing

Contributions are welcome! Please open an issue first to discuss what you'd like to change.

**Especially welcome:**
- New restaurant chain providers
- Bug fixes for existing provider scrapers (API hashes change when restaurants redeploy)
- UI/UX improvements

## License

[GPL-3.0](LICENSE)

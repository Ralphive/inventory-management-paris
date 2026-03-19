# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Factory Inventory Management System — full-stack demo with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database). Data is loaded from JSON files in `server/data/` at startup and reset on restart.

## Commands

### Backend
```bash
cd server && uv run python main.py        # Start API on port 8001
```

### Frontend
```bash
cd client && npm run dev                  # Start dev server on port 3000
cd client && npm run build                # Production build
```

### One-command startup
```bash
./scripts/start.sh
./scripts/stop.sh
```

### Backend Tests
```bash
cd tests && uv run pytest -v                                                      # All 51 tests
cd tests && uv run pytest backend/test_inventory.py -v                            # Single file
cd tests && uv run pytest backend/test_orders.py::TestOrderEndpoints::test_get_all_orders -v  # Single test
cd tests && uv run pytest --cov=../server --cov-report=html                       # With coverage
```

No frontend test suite is configured; use Playwright MCP tools for browser testing against `http://localhost:3000`.

## Architecture

### Data Flow
```
Vue component → useFilters composable → api.js (axios)
  → query params → FastAPI → apply_filters() → in-memory data
  → Pydantic validation → JSON response → ref/computed in component
```

### Filter System
Four global filters managed by the `useFilters.js` composable (singleton pattern):
- **Time Period** → `month` param (e.g. `2025-01`, `Q1-2025`)
- **Warehouse** → `warehouse` param (San Francisco, London, Tokyo)
- **Category** → `category` param (circuit boards, sensors, actuators, controllers, power supplies)
- **Order Status** → `status` param (delivered, shipped, processing, backordered)

Filter support per endpoint:
| Endpoint | warehouse | category | status | month |
|---|---|---|---|---|
| `/api/inventory` | ✓ | ✓ | | |
| `/api/orders` | ✓ | ✓ | ✓ | ✓ |
| `/api/dashboard/summary` | ✓ | ✓ | ✓ | ✓ |
| `/api/demand`, `/api/backlog` | | | | |
| `/api/spending/*` | | | | |

### Reactivity Pattern
Raw API data lives in `ref()`, derived values in `computed()`. Never mutate raw data directly — always re-fetch or derive via computed.

### Key Files
- `client/src/api.js` — Axios client; all API calls go through here
- `client/src/composables/useFilters.js` — Global filter state shared across all views
- `server/main.py` — All FastAPI endpoints and Pydantic models
- `server/mock_data.py` — Loads JSON files into memory at startup
- `tests/backend/conftest.py` — Pytest fixtures (FastAPI TestClient)

### Subagent Rules
- **MANDATORY**: Delegate any `.vue` file creation or significant modification to the `vue-expert` subagent
- Use `code-reviewer` after writing significant code
- Use `Explore` for codebase research across multiple files

### MCP Tools
- **GitHub operations**: Always use `mcp__github__*` tools (exception: local-only branches can use `git checkout -b`)
- **Browser testing**: Always use `mcp__playwright__*` tools

## Common Pitfalls

1. **v-for keys**: Use unique domain keys (`:key="item.sku"`), never array index
2. **Date validation**: Check `!isNaN(date.getTime())` before calling `.getMonth()`
3. **Pydantic models**: Update `server/main.py` models whenever JSON data structure changes
4. **Inventory has no time dimension** — don't pass `month` filter to `/api/inventory`
5. **Revenue targets**: $800K/month (single month), $9.6M YTD (all months)

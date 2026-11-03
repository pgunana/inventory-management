# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> ⚠️ **This repository and any fork you create are PUBLIC.** Do not commit credentials, internal hostnames, or private registry URLs. `client/.npmrc` pins the public npm registry and `client/package-lock.json` is gitignored — leave both in place.

## Stack

- **Frontend**: Vue 3 + Composition API + Vite — port 3000
- **Backend**: Python FastAPI — port 8001
- **Data**: JSON files in `server/data/` loaded into memory at startup via `server/mock_data.py`

## Commands

### Backend

```bash
cd server
uv run python main.py          # Start server (http://localhost:8001)
# OR with plain pip venv:
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8001
```

### Frontend

```bash
cd client
npm install
npm run dev      # http://localhost:3000
npm run build    # Output: client/dist/
```

### Tests

```bash
cd tests
uv run pytest backend/ -v              # All backend tests
uv run pytest backend/test_inventory.py -v   # Single test file
uv run pytest backend/ -k "test_name"  # Single test by name
```

### One-command startup

```bash
./scripts/start.sh   # starts both servers (macOS/Linux only)
./scripts/stop.sh
```

## Architecture

The app is a read-mostly dashboard — no user auth, no persistent writes (except purchase orders which live only in memory until restart).

**Data flow:**
1. Vue views call `client/src/api.js` (axios wrapper) with filter objects
2. `api.js` serializes filters to query params, hits FastAPI at `localhost:8001/api`
3. FastAPI's `apply_filters()` and `filter_by_month()` in `server/main.py` filter in-memory lists
4. Pydantic response models validate the output before it's returned
5. Vue views store raw API results in `ref()`s and expose filtered/derived data via `computed()`

**Filter system:** Four global filters (Time Period, Warehouse, Category, Order Status) live in `client/src/composables/` and are shared across all views. Passing `'all'` for any filter is treated as "no filter". The inventory endpoint does not support the `month` filter (no time dimension on inventory items).

**Frontend state pattern:** Raw data in `ref()` → derived values in `computed()` → never mutate computed properties, emit up to parent instead.

**Charts:** Custom SVG (no charting library). All chart calculations go in `computed()` properties.

## Key Files

| File | Purpose |
|---|---|
| `server/main.py` | All API endpoints + Pydantic models + filter helpers |
| `server/mock_data.py` | Loads JSON files, exports lists consumed by main.py |
| `server/data/*.json` | Source of truth for all data |
| `client/src/api.js` | Centralized axios API client |
| `client/src/App.vue` | Global styles and app shell |
| `client/src/views/*.vue` | One file per page (Dashboard, Inventory, Orders, etc.) |
| `client/src/composables/` | Shared filter state and reusable logic |
| `tests/backend/conftest.py` | pytest fixtures (FastAPI TestClient) |

## Subagents & Skills

- **vue-expert**: MANDATORY for creating or significantly modifying any `.vue` file
- **code-reviewer**: Use after writing significant code
- **Explore**: For codebase searches and structural questions
- **backend-api-test** skill: Use when writing/modifying tests in `tests/backend/`

## MCP Tools

- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for all GitHub operations (exception: local branch creation — use `git checkout -b`)
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing against `http://localhost:3000` (frontend) and `http://localhost:8001` (API)

## Common Pitfalls

1. **v-for keys**: Use `item.sku`, `item.id`, etc. — never array index
2. **Date parsing**: Validate with `!isNaN(date.getTime())` before calling `.getMonth()`
3. **Pydantic sync**: When changing `server/data/*.json` structure, update the Pydantic model in `main.py` to match
4. **Inventory has no month filter**: Don't add `month` param to `/api/inventory` calls
5. **Revenue goal constants**: $800K/month (single month), $9.6M YTD (all months)
6. **Data mutations**: Never mutate the in-memory lists — always filter onto a new list

## Code Style

- Always document non-obvious logic changes with comments

## Design System

- Colors: Slate/gray (`#0f172a`, `#64748b`, `#e2e8f0`)
- Status colors: green / blue / yellow / red
- Layouts: CSS Grid; component arrangement: Flexbox
- No emojis in UI

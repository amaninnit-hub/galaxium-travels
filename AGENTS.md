# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Project Structure

Full-stack interplanetary booking app: Python FastAPI backend + React/TypeScript frontend.
**All backend commands must be run from `booking_system_backend/`; all frontend commands from `booking_system_frontend/`.**

## Commands

### Backend
```bash
cd booking_system_backend
python server.py                          # runs on :8080
pytest                                    # all tests
pytest tests/test_rest.py::TestFlightsEndpoint::test_get_flights_empty  # single test
pytest tests/test_rest.py -k "test_name" # filter tests
```

### Frontend
```bash
cd booking_system_frontend
npm run dev       # dev server on :5173
npm run build     # tsc -b && vite build
npm run lint      # eslint
```

## Critical Architecture Notes

- **Dual protocol**: The backend exposes both REST (`/flights`, `/book`, `/bookings/{id}`, `/cancel/{id}`, `/register`, `/user`) and MCP (`/mcp`) from the same FastAPI app. MCP is mounted via `app.mount("/mcp", mcp_app)`.
- **MCP tools** in `server.py` call `SessionLocal()` directly (not the `get_db` dependency). REST endpoints use `Depends(get_db)`.
- **DB seeded on startup**: `seed()` runs in the lifespan. Tests monkeypatch `server.seed` to `lambda: None` to prevent seeding.
- **SQLite file**: `booking_system_backend/booking.db` — created automatically; not committed.
- **Frontend API URL**: configured via `VITE_API_URL` env var (copy `.env.example` to `.env`). Defaults to `http://localhost:8080`.

## Backend Patterns

- Service functions return `ModelOut | ErrorResponse` (never raise exceptions). REST endpoints return the object directly; MCP tools check `isinstance(result, ErrorResponse)` and `raise Exception(...)`.
- All Pydantic output schemas have `class Config: from_attributes = True`; use `.model_validate(orm_obj)` to convert SQLAlchemy models.
- Datetime stored as ISO string (`datetime.utcnow().isoformat()`), not a DateTime column — matches frontend string type.
- `book_flight` validates both `user_id` AND `name` match (name mismatch returns `NAME_MISMATCH` error).

## Frontend Patterns

- All shared types are in `src/types/index.ts` — keep frontend types in sync with backend schemas.
- API calls go through the single axios instance in `src/services/api.ts`. Use `isErrorResponse()` helper to distinguish success/error union returns.
- User session persisted in `localStorage` under key `galaxium_user` via `useUser` hook (`src/hooks/useUser.tsx`). Access user context only inside `UserProvider`.
- Custom Tailwind theme tokens: `space-dark`, `space-blue`, `cosmic-purple`, `nebula-pink`, `alien-green`, `solar-orange`, `star-white`, `space-gradient`, `cosmic-gradient`. Use these instead of raw hex values.
- TS strict mode with `noUnusedLocals`, `noUnusedParameters`, `verbatimModuleSyntax`, `erasableSyntaxOnly` — unused imports/params are compile errors.

## Testing Notes

- Tests use an **in-memory SQLite** database with `StaticPool`. `conftest.py` monkeypatches `db.SessionLocal` and `server.SessionLocal` to return the test session.
- Each test function gets a fresh DB (fixture scope is `function`).
- Test files must have the `sys.path.insert` at the top to resolve backend modules (no package install step).

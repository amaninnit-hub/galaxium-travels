# Galaxium Travels — System Architecture

High-level overview of the React/TypeScript frontend, Python FastAPI backend (REST + MCP), and SQLite database.

```mermaid
architecture-beta
    group frontend(cloud)[React / TypeScript · :5173]
    group backend(server)[Python FastAPI · :8080]
    group db_group(database)[SQLite]

    service browser(internet)[Browser]

    service pages(disk)[Pages\nFlights · MyBookings · Home] in frontend
    service hooks(disk)[useUser Hook\nlocalStorage] in frontend
    service api_svc(disk)[api.ts\nAxios Instance] in frontend

    service rest(server)[REST Layer\n/flights · /book\n/bookings · /cancel\n/register · /user] in backend
    service mcp(server)[MCP Layer\n/mcp\nfastmcp tools] in backend
    service services(server)[Service Layer\nflight · booking · user] in backend
    service orm(server)[SQLAlchemy ORM\nSessionLocal] in backend

    service sqlite(database)[booking.db\nusers · flights · bookings] in db_group

    browser:R --> L:pages
    pages:R --> L:api_svc
    hooks:R --> L:api_svc
    api_svc:R --> L:rest
    rest:B --> T:services
    mcp:B --> T:services
    services:B --> T:orm
    orm:R --> L:sqlite
```

## Components

### Frontend (`booking_system_frontend/`)

| Layer | Files | Responsibility |
|---|---|---|
| Pages | `Flights.tsx`, `MyBookings.tsx`, `Home.tsx` | Route-level views; compose components |
| Components | `FlightCard`, `BookingModal`, `BookingCard`, `UserIdentification`, common UI | Presentational and modal logic |
| Hook | `useUser.tsx` | User session via `UserContext` + `localStorage` key `galaxium_user` |
| API service | `api.ts` | Single Axios instance; base URL from `VITE_API_URL`; `isErrorResponse()` type-guard |
| Types | `types/index.ts` | Shared TypeScript types mirroring backend schemas |

### Backend (`booking_system_backend/`)

| Layer | Files | Responsibility |
|---|---|---|
| Entry point | `server.py` | Creates `FastMCP` + `FastAPI` apps; mounts MCP at `/mcp`; runs lifespan (init DB + seed) |
| REST endpoints | `server.py` (`@app.*`) | HTTP routes using `Depends(get_db)` |
| MCP tools | `server.py` (`@mcp.tool()`) | AI-agent tools; use `SessionLocal()` directly; raise on `ErrorResponse` |
| Service layer | `services/flight.py`, `services/booking.py`, `services/user.py` | Business logic; return `ModelOut \| ErrorResponse` (never raise) |
| ORM models | `models.py` | SQLAlchemy models: `User`, `Flight`, `Booking` |
| Schemas | `schemas.py` | Pydantic v2 schemas with `from_attributes = True` |
| Database | `db.py` | SQLite engine + `SessionLocal` + `get_db` dependency |
| Seed | `seed.py` | Populates initial flight data on startup |

### Database (`booking.db`)

| Table | Key columns |
|---|---|
| `users` | `user_id` PK, `name`, `email` (unique) |
| `flights` | `flight_id` PK, `origin`, `destination`, `departure_time`, `arrival_time`, `price`, `seats_available` |
| `bookings` | `booking_id` PK, `user_id` FK, `flight_id` FK, `status`, `booking_time` (ISO string) |

### Dual-protocol design

The same FastAPI app serves both protocols simultaneously:

- **REST** — consumed by the React frontend via Axios
- **MCP** — mounted at `/mcp`, consumed by AI agents via `fastmcp`; shares the same service layer and SQLite database

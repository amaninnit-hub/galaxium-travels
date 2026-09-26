# 🚀 Galaxium Travels — Application Walkthrough

Welcome to **Galaxium Travels**, an interplanetary flight booking app. This document explains how the whole system hangs together in plain English. No prior context needed.

---

## What Does the App Do?

Users can browse flights between planets (Earth, Mars, Moon, Venus, Jupiter, etc.), register/sign in with their name and email, book seats on flights, and cancel bookings. There is no password — identity is just a name + email pair. The system is intentionally lightweight to focus on the booking flow.

---

## Big-Picture Architecture

```
┌────────────────────────────┐        HTTP        ┌────────────────────────┐
│  React Frontend            │◄──────────────────►│  FastAPI Backend       │
│  booking_system_frontend/  │  REST + JSON        │  booking_system_backend│
│  localhost:5173            │                     │  localhost:8080        │
└────────────────────────────┘                     └───────────┬────────────┘
                                                               │
                                           ┌───────────────────┼───────────────────┐
                                           │                   │                   │
                                      REST API            MCP API            SQLite DB
                                     /flights           /mcp endpoint        booking.db
                                     /book              (AI agent use)
                                     /cancel/{id}
                                     /bookings/{id}
                                     /register
                                     /user
```

The backend is unusual: it serves **two protocols from a single Python process**. REST endpoints are used by the React frontend; the `/mcp` endpoint uses the [Model Context Protocol](https://modelcontextprotocol.io/) so that AI assistants can also book flights programmatically. Both speak to the same SQLite database.

---

## Backend Deep Dive

### Entry Point: [`server.py`](../../booking_system_backend/server.py)

This is the only file you need to run to start the backend. It does four things:

1. **Creates the MCP server** — defines AI-callable tools (`list_flights`, `book_flight`, `get_bookings`, `cancel_booking`, `register_user`, `get_user_id`) using the `FastMCP` library.
2. **Creates the FastAPI app** — adds CORS middleware (allows all origins in dev), and registers REST routes.
3. **Runs the lifespan** — on startup, calls `init_db()` to create tables, then `seed()` to populate demo data.
4. **Mounts MCP into FastAPI** — `app.mount("/mcp", mcp_app)` so one process handles both.

> **Key rule:** MCP tool functions call `SessionLocal()` directly and manage the DB session themselves. REST endpoint functions get the session injected via `Depends(get_db)`. Both patterns are intentional — don't mix them.

### Database Layer

| File | What it does |
|------|-------------|
| [`models.py`](../../booking_system_backend/models.py) | SQLAlchemy ORM models: `User`, `Flight`, `Booking` |
| [`db.py`](../../booking_system_backend/db.py) | Creates the SQLite engine, `SessionLocal` factory, and `get_db` dependency |
| [`seed.py`](../../booking_system_backend/seed.py) | Clears and re-populates the DB with 10 users, 10 flights, 20 sample bookings |

The database file (`booking.db`) is created automatically in `booking_system_backend/` when the server starts. It is **not committed to git**.

Dates are stored as ISO strings (e.g. `"2099-01-01T09:00:00Z"`) in plain `String` columns — not datetime columns. This matches what the frontend TypeScript types expect.

### Schemas (API contracts)

[`schemas.py`](../../booking_system_backend/schemas.py) defines the Pydantic models used for request validation and response serialisation:

- `FlightOut`, `BookingOut`, `UserOut` — output shapes, all have `from_attributes = True` so they work with SQLAlchemy ORM objects via `.model_validate(orm_obj)`
- `BookingRequest`, `UserRegistration` — input shapes for POST bodies
- `ErrorResponse` — returned instead of raising HTTP exceptions when business rules fail (e.g. flight not found, email already taken)

### Service Layer

Business logic lives in [`services/`](../../booking_system_backend/services/):

| File | Responsibility |
|------|---------------|
| [`flight.py`](../../booking_system_backend/services/flight.py) | `list_flights` — returns all flights |
| [`user.py`](../../booking_system_backend/services/user.py) | `register_user`, `get_user` — create/look up users |
| [`booking.py`](../../booking_system_backend/services/booking.py) | `book_flight`, `cancel_booking`, `get_bookings` — core booking logic |

**All service functions return `ModelOut | ErrorResponse`** — they never raise exceptions. The REST layer returns the result directly; the MCP layer checks `isinstance(result, ErrorResponse)` and raises a Python `Exception` if it is one (MCP clients expect exceptions for errors).

`book_flight` in [`services/booking.py`](../../booking_system_backend/services/booking.py:7) validates:
1. Flight exists
2. Seats are available
3. User exists **and** the supplied `name` matches the registered name (prevents booking under someone else's identity)

---

## Frontend Deep Dive

### Entry Point: [`App.tsx`](../../booking_system_frontend/src/App.tsx)

Wraps the app in `BrowserRouter` and `UserProvider`, then declares three routes:

| Path | Component | Purpose |
|------|-----------|---------|
| `/` | `Home` | Landing page / hero |
| `/flights` | `Flights` | Browse & book flights |
| `/bookings` | `MyBookings` | View & cancel user's bookings |

### User Session: [`useUser.tsx`](../../booking_system_frontend/src/hooks/useUser.tsx)

A React Context + `localStorage` combination. When a user signs in (or registers), their `{ user_id, name, email }` object is saved to `localStorage` under the key `galaxium_user`. On next page load, it is read back so the session persists across refreshes. `useUser()` can only be called inside a component that is a descendant of `<UserProvider>`.

### API Layer: [`services/api.ts`](../../booking_system_frontend/src/services/api.ts)

A single `axios` instance points at `VITE_API_URL` (defaults to `http://localhost:8080`). All network calls go through this file. The `isErrorResponse()` helper checks `response.success === false` to distinguish an `ErrorResponse` from a real data object — useful because the backend returns HTTP 200 even for business-logic errors.

### TypeScript Types: [`types/index.ts`](../../booking_system_frontend/src/types/index.ts)

All shared data shapes live here. These must stay in sync with the backend Pydantic schemas in [`schemas.py`](../../booking_system_backend/schemas.py). If you add a field to the backend, add it here too.

### Styling

The app uses [Tailwind CSS](https://tailwindcss.com/) with a custom space theme defined in [`tailwind.config.js`](../../booking_system_frontend/tailwind.config.js). Custom colour tokens:

| Token | Hex | Use |
|-------|-----|-----|
| `space-dark` | `#030712` | Page background |
| `space-blue` | `#0A1929` | Card backgrounds |
| `cosmic-purple` | `#6366F1` | Primary accents |
| `nebula-pink` | `#EC4899` | Gradient highlights |
| `alien-green` | `#10B981` | Success states |
| `solar-orange` | `#F59E0B` | Warnings |
| `star-white` | `#F9FAFB` | Body text |

Always use these tokens instead of raw hex values in JSX.

---

## Request Lifecycle Example — Booking a Flight

1. User clicks **Book Now** on a [`FlightCard`](../../booking_system_frontend/src/components/flights/FlightCard.tsx) in the [`Flights`](../../booking_system_frontend/src/pages/Flights.tsx) page.
2. If not logged in, the [`UserIdentification`](../../booking_system_frontend/src/components/user/UserIdentification.tsx) modal opens to register/sign in.
3. Once identified, the [`BookingModal`](../../booking_system_frontend/src/components/bookings/BookingModal.tsx) appears for confirmation.
4. On confirm, [`bookFlight()`](../../booking_system_frontend/src/services/api.ts:77) in `api.ts` sends `POST /book` with `{ user_id, name, flight_id }`.
5. Backend [`book_flight_endpoint`](../../booking_system_backend/server.py:146) receives the request and calls [`services/booking.book_flight()`](../../booking_system_backend/services/booking.py:7).
6. The service validates the flight, checks seats, and checks the user name matches. On success it decrements `seats_available`, creates a `Booking` row, and returns `BookingOut`.
7. The frontend receives the response, checks `isErrorResponse()`, shows a toast notification, and reloads the flights list to reflect the updated seat count.

---

## Testing

Backend tests live in [`tests/`](../../booking_system_backend/tests/). The test setup in [`conftest.py`](../../booking_system_backend/tests/conftest.py) creates a fresh **in-memory SQLite database** for every test function, monkeypatches both `db.SessionLocal` and `server.SessionLocal` to return the test session, and patches `server.seed` to do nothing (so the database stays clean). This means each test starts from an empty database and builds only the data it needs.

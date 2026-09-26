# 🎯 Good First Tasks

Five concrete starting tasks ranked from easiest to hardest. Each one teaches a different layer of the codebase. References to exact files and line numbers are included so you know exactly where to start.

---

## Task 1 — Add a `GET /flights/{flight_id}` Endpoint ⭐ (Easiest)

**What:** The API currently only supports listing all flights (`GET /flights`). Add a new endpoint that returns a single flight by its ID, or an `ErrorResponse` if it doesn't exist.

**Why it's a good first task:** It's small, self-contained, and follows patterns already in the codebase. You'll learn the service → schema → REST endpoint pipeline end-to-end.

**Files to change:**

1. [`booking_system_backend/services/flight.py`](../../booking_system_backend/services/flight.py) — add a `get_flight(db, flight_id)` function that returns `FlightOut | ErrorResponse`. Follow the same pattern used in [`services/booking.py`](../../booking_system_backend/services/booking.py:7).

2. [`booking_system_backend/server.py`](../../booking_system_backend/server.py) — add a new route:
   ```python
   @app.get("/flights/{flight_id}", response_model=Union[FlightOut, ErrorResponse], tags=["Flights"])
   def get_flight_endpoint(flight_id: int, db: Session = Depends(get_db)):
       ...
   ```

3. [`booking_system_backend/tests/test_rest.py`](../../booking_system_backend/tests/test_rest.py) — add tests to `TestFlightsEndpoint`: one for a found flight, one for a not-found flight.

**How to verify:** Run `pytest tests/test_rest.py` and visit http://localhost:8080/docs to see the new endpoint in Swagger UI.

---

## Task 2 — Add a `flight` Field to `BookingCard` ⭐⭐ (Easy–Medium)

**What:** The [`BookingCard`](../../booking_system_frontend/src/components/bookings/BookingCard.tsx) component already receives a `flight` prop (see [`MyBookings.tsx`](../../booking_system_frontend/src/pages/MyBookings.tsx:79)), but it may not display the full flight details (origin, destination, departure time). Improve it to show the route clearly, e.g. "Earth → Mars, departs 2099-01-01".

**Why it's a good first task:** You'll learn how to use the existing TypeScript types and the Tailwind space-theme tokens, and get comfortable with how props flow from a page into a child component.

**Files to change:**

1. [`booking_system_frontend/src/components/bookings/BookingCard.tsx`](../../booking_system_frontend/src/components/bookings/BookingCard.tsx) — update the JSX to display `flight.origin`, `flight.destination`, and formatted `flight.departure_time`.

2. [`booking_system_frontend/src/utils/formatters.ts`](../../booking_system_frontend/src/utils/formatters.ts) — there may already be date-formatting helpers here. Use or extend them rather than writing `new Date()` logic inline. The project already has [`date-fns`](../../booking_system_frontend/package.json:17) installed.

**Rules to follow:**
- Use only the theme tokens listed in [`tailwind.config.js`](../../booking_system_frontend/tailwind.config.js:9) for colours.
- Don't add unused imports — the TypeScript build will fail ([`tsconfig.app.json`](../../booking_system_frontend/tsconfig.app.json) has `noUnusedLocals: true`).

**How to verify:** Run `npm run build` — it must compile with no errors. Then visually check "My Bookings" in the browser.

---

## Task 3 — Add a `seats_available` Badge to `FlightCard` ⭐⭐⭐ (Medium)

**What:** Show a visual urgency indicator on each flight card based on seat availability: e.g. green for ≥5 seats, yellow for 2–4 seats, red for 1 seat ("Last seat!"), and a "Sold Out" state for 0 seats with the Book button disabled.

**Why it's a good first task:** You'll learn the component-level rendering patterns and how to handle conditional UI states. It also connects to a real UX concern in booking apps.

**Files to change:**

1. [`booking_system_frontend/src/components/flights/FlightCard.tsx`](../../booking_system_frontend/src/components/flights/FlightCard.tsx) — add a badge/label that reads from the `flight.seats_available` field (already in the [`Flight`](../../booking_system_frontend/src/types/index.ts:3) type). Disable the book button when `seats_available === 0`.

2. [`booking_system_frontend/src/components/common/Button.tsx`](../../booking_system_frontend/src/components/common/Button.tsx) — check if a `disabled` prop is already supported. If not, add it.

**Colour guidance:** Use `alien-green` for plenty, `solar-orange` for low, `nebula-pink` for last seat, `space-blue` (greyed out) for sold out.

**How to verify:** `npm run build` passes. In the browser, use Swagger UI at http://localhost:8080/docs to temporarily set `seats_available` to 0 or 1 on a flight (by calling `POST /book` repeatedly) and confirm the badge updates.

---

## Task 4 — Add a `GET /flights/{flight_id}` MCP Tool ⭐⭐⭐ (Medium)

**What:** The MCP server (used by AI agents) has `list_flights` but no way to look up a single flight. Add a `get_flight` MCP tool that mirrors the REST endpoint from Task 1.

**Why it's a good first task:** You'll learn the dual-protocol pattern that makes this codebase unique — specifically how MCP tools differ from REST endpoints in how they handle errors (raising `Exception` vs returning `ErrorResponse`).

**Prerequisite:** Complete Task 1 first so `services/flight.get_flight()` exists.

**File to change:**

[`booking_system_backend/server.py`](../../booking_system_backend/server.py:19) — add a new `@mcp.tool()` decorated function below the existing MCP tools. Follow the exact pattern of `cancel_booking` at line 57:

```python
@mcp.tool()
def get_flight(flight_id: int) -> FlightOut:
    """Get a single flight by its ID.
    Returns flight details or raises an error if not found."""
    db = SessionLocal()
    try:
        result = flight.get_flight(db, flight_id)
        if isinstance(result, ErrorResponse):
            raise Exception(result.details or result.error)
        return result
    finally:
        db.close()
```

**How to verify:** Add a test in [`tests/test_services.py`](../../booking_system_backend/tests/test_services.py) or test via the MCP endpoint using an AI assistant configured to point at `http://localhost:8080/mcp`.

---

## Task 5 — Add Price Filtering to the Flights Page ⭐⭐⭐⭐ (Harder)

**What:** The [`Flights`](../../booking_system_frontend/src/pages/Flights.tsx) page has a search bar that filters by origin/destination. Extend it with a price range filter (min/max inputs). The [`FlightFilters`](../../booking_system_frontend/src/types/index.ts:51) type already has `minPrice` and `maxPrice` fields — they just aren't wired up yet.

**Why it's a good first task:** You'll work across multiple frontend layers: types (already defined), state management in a page component, a reusable `Input` component, and the filter logic itself. It's a realistic frontend feature that mirrors what you'd do on a real project.

**Files to change:**

1. [`booking_system_frontend/src/pages/Flights.tsx`](../../booking_system_frontend/src/pages/Flights.tsx:16) — add `minPrice` and `maxPrice` to state. Update the filter `useEffect` at line 29 to also apply price range filtering. Add two number inputs to the search/filter bar JSX.

2. [`booking_system_frontend/src/components/common/Input.tsx`](../../booking_system_frontend/src/components/common/Input.tsx) — check if this component supports `type="number"`. If not, extend it (or use a plain `<input type="number" />` inline with the same Tailwind classes as the existing search input).

**Hints:**
- Prices in the data are integers (e.g. `1000000` = 1,000,000 credits). Display them with the existing formatter in [`utils/formatters.ts`](../../booking_system_frontend/src/utils/formatters.ts).
- An empty min/max input should mean "no filter" (i.e. don't filter on that bound).
- The filter count badge (`{filteredFlights.length} of {flights.length} flights`) at line 119 will update automatically since it already reads from `filteredFlights`.

**How to verify:** `npm run lint` and `npm run build` both pass cleanly. In the browser, type a max price lower than all flights and confirm the list goes empty.

---

## Quick Reference

| # | Task | Difficulty | Teaches |
|---|------|-----------|---------|
| 1 | Add `GET /flights/{flight_id}` REST endpoint | ⭐ | Backend service + REST layer |
| 2 | Show flight details on `BookingCard` | ⭐⭐ | Frontend components + types |
| 3 | Seat availability badge on `FlightCard` | ⭐⭐⭐ | Conditional UI + component props |
| 4 | Add `get_flight` MCP tool | ⭐⭐⭐ | Dual-protocol pattern |
| 5 | Price range filtering on Flights page | ⭐⭐⭐⭐ | Full frontend feature slice |

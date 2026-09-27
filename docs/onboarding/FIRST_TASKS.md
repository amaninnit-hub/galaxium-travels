# Good First Tasks — Galaxium Travels

> Ranked from easiest (#1) to most involved (#5).  
> Each task is self-contained and safe to attempt without breaking other features.
> All file references include exact line numbers — open the file and go straight to the code.

---

## Task 1 — Fix Email Input Type Validation · ⭐ Easy

### Why it matters
The user registration form currently uses `type="text"` for the email field instead of `type="email"`.
This means browsers skip their built-in email format check, so users can accidentally submit
garbage values like `"hello"` or `"foo@"`. One-line fix, immediate user-experience win.

### Where to work
- [`booking_system_frontend/src/components/user/UserIdentification.tsx`](../../booking_system_frontend/src/components/user/UserIdentification.tsx) — line ~99, the email `<Input>` element

### Steps
1. Open `UserIdentification.tsx`.
2. Find the email Input component (line 99) and change `type="text"` → `type="email"`.
3. Open the app in a browser, try typing `"notanemail"` and submit — browser should show a native validation tooltip.
4. Verify the form still submits correctly with a proper email address.
5. Run `npm run lint` from `booking_system_frontend/` — no new errors expected.

### Acceptance criteria
- Email input has `type="email"`.
- Browser shows a native validation error for malformed addresses.
- Form still works end-to-end with a valid email.
- Zero TypeScript / lint errors.

---

## Task 2 — Implement Price Range Filtering for Flights · ⭐ Easy

### Why it matters
The `FlightFilters` type already declares `minPrice` and `maxPrice` fields
([`src/types/index.ts:51-57`](../../booking_system_frontend/src/types/index.ts)), but the Flights page
never reads them — the fields are stubbed out, waiting to be wired up. Adding price filter inputs
and hooking them into the existing filter effect is a complete, visible feature with no risk to
existing functionality.

### Where to work
- [`booking_system_frontend/src/types/index.ts:51-57`](../../booking_system_frontend/src/types/index.ts) — `FlightFilters` interface (already defined, no changes needed)
- [`booking_system_frontend/src/pages/Flights.tsx:18-42`](../../booking_system_frontend/src/pages/Flights.tsx) — filtering `useEffect` block
- [`booking_system_frontend/src/pages/Flights.tsx:96-124`](../../booking_system_frontend/src/pages/Flights.tsx) — filter UI card

### Steps
1. In `Flights.tsx`, add two new state variables: `minPrice` and `maxPrice` (both `number | undefined`, initialize to `undefined`).
2. In the filter UI card (lines 96-124), add two `<Input>` elements — one for min price, one for max.
3. In the filtering `useEffect` (lines 28-42), add conditions:
   - If `minPrice` is set, exclude flights where `flight.price < minPrice`.
   - If `maxPrice` is set, exclude flights where `flight.price > maxPrice`.
4. Style the inputs consistently with the existing origin/destination fields.
5. Test: enter a low max price → list shrinks; clear it → full list returns.

### Acceptance criteria
- Two price input fields appear in the filter section.
- Flights outside the entered range are hidden from the list.
- Works with only min set, only max set, or both.
- Inputs accept numbers only (no negative values).
- `npm run build` passes with no TypeScript errors.

---

## Task 3 — Wire Up the Existing `error` Prop on the Input Component · ⭐⭐ Medium

### Why it matters
[`src/components/common/Input.tsx`](../../booking_system_frontend/src/components/common/Input.tsx) already has a fully-implemented `error` prop with a red-border style and
error-message display (lines 4-30) — but nothing in the app actually passes a value to it.
Hooking this up in the registration form replaces the current catch-all toast with precise,
inline field-level errors that guide the user to exactly what they got wrong.

### Where to work
- [`booking_system_frontend/src/components/common/Input.tsx:4-30`](../../booking_system_frontend/src/components/common/Input.tsx) — existing `error` prop (read-only reference, no changes needed)
- [`booking_system_frontend/src/components/user/UserIdentification.tsx:13-122`](../../booking_system_frontend/src/components/user/UserIdentification.tsx) — form component to update

### Steps
1. In `UserIdentification.tsx`, add a new state object:
   ```ts
   const [errors, setErrors] = useState<{ name?: string; email?: string }>({});
   ```
2. In `handleSubmit`, replace the single toast validation with an errors-object approach:
   - If name is empty → `errors.name = "Name is required"`.
   - If email fails a basic regex (`/^\S+@\S+\.\S+$/`) → `errors.email = "Enter a valid email"`.
   - Only submit if both checks pass.
3. Pass `error={errors.name}` to the name `<Input>` (line ~88) and `error={errors.email}` to the email `<Input>` (line ~99).
4. Clear the relevant error when the user starts typing in each field (`onChange` handler).
5. Test: submit the empty form → both inline errors appear; fix a field → its error disappears.

### Acceptance criteria
- Name and email fields display inline error messages below them.
- Errors clear as soon as the user edits the field.
- No toast fired for validation errors that are now shown inline.
- Existing end-to-end flow (register → view bookings) still works.
- `npm run build` passes with no TypeScript errors.

---

## Task 4 — Extract Flight Filtering into a Reusable Service Function · ⭐⭐ Medium

### Why it matters
All flight-filter logic currently lives embedded inside `Flights.tsx`. Moving it to a
standalone function in [`src/services/api.ts`](../../booking_system_frontend/src/services/api.ts) makes it independently testable and reusable if
other parts of the app (e.g., a booking confirmation page) ever need the same logic. This is a
classic "extract function" refactor — good practice for any growing codebase.

### Where to work
- [`booking_system_frontend/src/services/api.ts:37-45`](../../booking_system_frontend/src/services/api.ts) — add the new function here
- [`booking_system_frontend/src/types/index.ts:51-57`](../../booking_system_frontend/src/types/index.ts) — `FlightFilters` interface (input type for the function)
- [`booking_system_frontend/src/pages/Flights.tsx:28-42`](../../booking_system_frontend/src/pages/Flights.tsx) — replace inline logic with a call to the new function

### Steps
1. In `api.ts`, export a new pure function:
   ```ts
   export function filterFlights(flights: Flight[], filters: FlightFilters): Flight[]
   ```
2. Implement: loop over flights and apply `origin`, `destination`, `searchTerm`, `minPrice`, `maxPrice`
   checks (case-insensitive string matching, skip checks where the filter value is `undefined`/empty).
3. In `Flights.tsx`, import and call `filterFlights(allFlights, { origin, destination, searchTerm, minPrice, maxPrice })` inside the effect, replacing the inline logic.
4. (Bonus) Add a test file `src/services/api.test.ts` with at least 3 cases: no filters returns all, price filter narrows list, empty input returns all.
5. Run `npm run build` and confirm zero errors.

### Acceptance criteria
- `filterFlights` is exported from `api.ts` and typed with `Flight[]` / `FlightFilters`.
- `Flights.tsx` no longer contains inline filter conditions.
- All existing filter behaviour works identically in the browser.
- Function handles edge cases: undefined filters, empty arrays, mixed case.
- `npm run build` and `npm run lint` pass cleanly.

---

## Task 5 — Add Query-Parameter Filtering to the `GET /flights` Backend Endpoint · ⭐⭐⭐ Medium-Hard

### Why it matters
Right now `GET /flights` always returns every flight in the database — the frontend filters the
full list in-browser. Moving filtering to the database means only matching rows travel over the
wire, which scales better and is the architecturally correct place for this logic. This is a
full-stack task that touches the backend service layer, REST endpoint, and optionally the frontend
API call — a great way to learn both sides of the codebase.

### Where to work
- [`booking_system_backend/services/flight.py`](../../booking_system_backend/services/flight.py) — add a new `filter_flights` service function
- [`booking_system_backend/server.py:139-142`](../../booking_system_backend/server.py) — existing `/flights` endpoint (add optional query params)
- [`booking_system_backend/tests/test_rest.py`](../../booking_system_backend/tests/test_rest.py) — add 2-3 new test cases

### Steps
1. In `services/flight.py`, add:
   ```python
   def filter_flights(
       db: Session,
       origin: str | None = None,
       destination: str | None = None,
       min_price: int | None = None,
       max_price: int | None = None,
   ) -> list[FlightOut]:
   ```
   Use SQLAlchemy `.filter()` calls — e.g., `Flight.price >= min_price` — and return a list of `FlightOut` objects.
2. In `server.py`, update the `/flights` route signature to accept optional query params:
   ```python
   @app.get("/flights")
   def get_flights(origin: str | None = None, ..., db: Session = Depends(get_db)):
   ```
3. Inside the endpoint, call `filter_flights(db, origin, ...)` if any filter is provided, otherwise fall back to `list_flights(db)`.
4. Check backward compatibility: `GET /flights` with no params must still return all flights.
5. Verify the new params appear in Swagger UI at `http://localhost:8080/docs`.
6. Add test cases in `test_rest.py`: filter by price returns subset; no filter returns all; out-of-range filter returns empty list.
7. Run `pytest` from `booking_system_backend/` — all tests green.

### Acceptance criteria
- `GET /flights?min_price=500000` returns only flights at or above that price.
- `GET /flights` (no params) still returns all flights — no regression.
- New params documented in `/docs` Swagger UI.
- All pre-existing tests pass; at least 2 new tests added.
- `FlightOut` schema unchanged (no breaking change to consumers).

# `book_flight` — End-to-End Sequence

Full flow from the user clicking **Book** in the browser through to the confirmation returned by the backend, including all validation branches.

```mermaid
sequenceDiagram
    autonumber

    actor User as User (Browser)
    participant Flights as Flights.tsx
    participant UserModal as UserIdentification<br/>Modal
    participant BookModal as BookingModal
    participant Hook as useUser Hook<br/>(localStorage)
    participant API as api.ts<br/>(Axios)
    participant REST as FastAPI REST<br/>POST /book
    participant SVC as booking.py<br/>book_flight()
    participant DB as SQLite<br/>(booking.db)

    User->>Flights: Clicks "Book" on a FlightCard

    alt User not identified
        Flights->>UserModal: Opens UserIdentification modal
        UserModal->>API: POST /register  OR  GET /user?name&email
        API->>REST: HTTP request
        REST->>SVC: user.register_user() / user.get_user()
        SVC->>DB: INSERT users / SELECT users
        DB-->>SVC: User row
        SVC-->>REST: UserOut | ErrorResponse
        REST-->>API: JSON response
        API-->>UserModal: User | ErrorResponse

        alt Registration / lookup successful
            UserModal->>Hook: setUser(user) → persists to localStorage
            UserModal-->>Flights: onSuccess callback
            Flights->>BookModal: Opens BookingModal
        else Error (duplicate email, not found, …)
            UserModal-->>User: Shows error toast
        end
    else User already identified
        Flights->>BookModal: Opens BookingModal directly
    end

    BookModal->>Hook: reads user (user_id, name)
    User->>BookModal: Confirms booking

    BookModal->>API: bookFlight({ user_id, name, flight_id })
    API->>REST: POST /book  { user_id, name, flight_id }

    REST->>SVC: booking.book_flight(db, user_id, name, flight_id)

    SVC->>DB: SELECT flights WHERE flight_id = ?
    DB-->>SVC: Flight row (or null)

    alt Flight not found
        SVC-->>REST: ErrorResponse FLIGHT_NOT_FOUND
        REST-->>API: 200 { success:false, error_code:"FLIGHT_NOT_FOUND" }
        API-->>BookModal: ErrorResponse
        BookModal-->>User: Shows error toast
    else Flight found
        SVC->>SVC: Check seats_available >= 1

        alt No seats available
            SVC-->>REST: ErrorResponse NO_SEATS_AVAILABLE
            REST-->>API: 200 { success:false, error_code:"NO_SEATS_AVAILABLE" }
            API-->>BookModal: ErrorResponse
            BookModal-->>User: Shows error toast
        else Seats available
            SVC->>DB: SELECT users WHERE user_id = ? AND name = ?
            DB-->>SVC: User row (or null)

            alt User not found
                SVC-->>REST: ErrorResponse USER_NOT_FOUND
                REST-->>API: 200 { success:false, error_code:"USER_NOT_FOUND" }
                API-->>BookModal: ErrorResponse
                BookModal-->>User: Shows error toast
            else user_id found but name mismatch
                SVC-->>REST: ErrorResponse NAME_MISMATCH
                REST-->>API: 200 { success:false, error_code:"NAME_MISMATCH" }
                API-->>BookModal: ErrorResponse
                BookModal-->>User: Shows error toast
            else User valid
                SVC->>DB: UPDATE flights SET seats_available = seats_available - 1
                SVC->>DB: INSERT bookings (user_id, flight_id, "booked", booking_time)
                DB-->>SVC: New Booking row
                SVC-->>REST: BookingOut (booking_id, user_id, flight_id, status, booking_time)
                REST-->>API: 200 BookingOut JSON
                API-->>BookModal: Booking object
                BookModal-->>User: Shows success toast + confirmation
                BookModal->>Flights: onSuccess callback
                Flights->>API: getFlights()
                API->>REST: GET /flights
                REST-->>API: Updated FlightOut[] (decremented seats)
                API-->>Flights: Updated flight list
                Flights-->>User: Re-renders grid with new seat count
            end
        end
    end
```

## Validation order in `book_flight()`

The service layer enforces checks in this exact sequence — an early failure short-circuits without touching the database further:

1. **Flight exists** — `FLIGHT_NOT_FOUND` if no matching `flight_id`
2. **Seats available** — `NO_SEATS_AVAILABLE` if `seats_available < 1`
3. **User exists + name matches** — `USER_NOT_FOUND` or `NAME_MISMATCH`
4. **Write** — decrement `seats_available`, insert `Booking` row, commit

## Key implementation notes

- The REST endpoint (`POST /book`) uses `Depends(get_db)` and returns `BookingOut | ErrorResponse` — it never raises an HTTP exception.
- The MCP tool (`@mcp.tool() book_flight`) calls `SessionLocal()` directly and converts an `ErrorResponse` into a raised `Exception` for the AI agent.
- `booking_time` is stored as an ISO 8601 string (`datetime.utcnow().isoformat()`), not a native `DateTime` column.
- After a successful booking the frontend reloads the full flight list so the seat count is always fresh.

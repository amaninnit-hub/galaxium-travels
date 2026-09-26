# 🛠️ Local Setup Guide

Step-by-step instructions to get Galaxium Travels running on your machine from scratch. Follow the steps in order.

---

## Prerequisites

Install these tools before you begin. Check with the version commands to confirm you have the right versions.

| Tool | Minimum version | Check command |
|------|----------------|---------------|
| Python | 3.8+ | `python --version` |
| pip | any recent | `pip --version` |
| Node.js | 18+ | `node --version` |
| npm | comes with Node | `npm --version` |
| git | any | `git --version` |

Download links if needed:
- Python: https://www.python.org/downloads/
- Node.js (includes npm): https://nodejs.org/

---

## Step 1 — Clone the Repository

```bash
git clone <repo-url>
cd galaxium-travels
```

---

## Step 2 — Set Up the Backend

All backend commands must be run from the `booking_system_backend/` directory.

### 2a. Create and activate a Python virtual environment

A virtual environment keeps this project's Python packages separate from everything else on your machine.

**macOS / Linux:**
```bash
cd booking_system_backend
python -m venv .venv
source .venv/bin/activate
```

**Windows (PowerShell):**
```powershell
cd booking_system_backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

You should see `(.venv)` appear at the start of your terminal prompt.

### 2b. Install Python dependencies

```bash
pip install -r requirements.txt
```

This installs: FastAPI, FastMCP, Uvicorn, SQLAlchemy, Pydantic, pytest, httpx, and a few more. See [`requirements.txt`](../../booking_system_backend/requirements.txt) for the full list.

### 2c. Start the backend server

```bash
python server.py
```

Expected output:
```
Database seeded with elaborate demo data!
INFO:     Started server process [...]
INFO:     Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
```

On first run, SQLite creates `booking_system_backend/booking.db` automatically. The seed function populates it with 10 users, 10 flights, and 20 sample bookings. **The database is re-seeded (wiped and re-populated) every time the server starts.**

> **Note:** Leave this terminal open. Open a new terminal for the frontend.

### Verify the backend is working

Open your browser at:
- **API health check:** http://localhost:8080/
  - Should return `{"status":"OK"}`
- **Interactive API docs (Swagger UI):** http://localhost:8080/docs
  - You can call every endpoint directly from the browser here — great for exploring.

---

## Step 3 — Set Up the Frontend

All frontend commands must be run from the `booking_system_frontend/` directory.

### 3a. Configure the API URL

```bash
cd booking_system_frontend
cp .env.example .env
```

The default content of `.env` is:
```
VITE_API_URL=http://localhost:8080
```

This points the React app at the backend you just started. You don't need to change this for local development.

> **Why `VITE_`?** Vite (the frontend build tool) only exposes environment variables that start with `VITE_` to the browser. See [`src/services/api.ts`](../../booking_system_frontend/src/services/api.ts:13) where it is read.

### 3b. Install Node dependencies

```bash
npm install
```

This reads [`package.json`](../../booking_system_frontend/package.json) and downloads all packages into `node_modules/`. It will take a minute.

### 3c. Start the development server

```bash
npm run dev
```

Expected output:
```
  VITE v7.x.x  ready in Xms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
```

Open http://localhost:5173/ in your browser. You should see the Galaxium Travels home page with an animated starfield.

---

## Step 4 — Run the Tests

Tests are backend-only (pytest). Make sure your virtual environment is still activated.

```bash
cd booking_system_backend
pytest
```

Expected output ends with something like:
```
============================= X passed in X.XXs =============================
```

To run a single test:
```bash
pytest tests/test_rest.py::TestFlightsEndpoint::test_get_flights_empty
```

To run tests that match a name pattern:
```bash
pytest tests/test_rest.py -k "cancel"
```

> **How the tests work:** Each test gets a fresh in-memory database. The test setup in [`conftest.py`](../../booking_system_backend/tests/conftest.py) patches `SessionLocal` to use this test database and prevents the `seed()` function from running. This means tests never touch `booking.db`.

---

## Step 5 — Run the Frontend Lint and Build

To check the frontend for TypeScript and lint errors:

```bash
cd booking_system_frontend
npm run lint      # ESLint check
npm run build     # TypeScript compile + Vite production build
```

If `npm run build` succeeds, it outputs files to `booking_system_frontend/dist/`. A successful build means there are no TypeScript errors — this is the project's frontend "test".

> **Strict TypeScript:** The project has `noUnusedLocals` and `noUnusedParameters` enabled in [`tsconfig.app.json`](../../booking_system_frontend/tsconfig.app.json). Unused imports or variables are **compile errors**, not just warnings. Remove them or you will break the build.

---

## Summary of Running Services

| Service | Command | URL |
|---------|---------|-----|
| Backend API | `python server.py` | http://localhost:8080 |
| Swagger UI | *(backend must be running)* | http://localhost:8080/docs |
| MCP endpoint | *(backend must be running)* | http://localhost:8080/mcp |
| Frontend dev | `npm run dev` | http://localhost:5173 |

---

## Common Problems

### `python: command not found` / `python3` on macOS
Some systems use `python3` instead of `python`. Try: `python3 server.py` and `python3 -m venv .venv`.

### Port already in use
- Backend 8080: Another process is using port 8080. Kill it or change the port in [`server.py`](../../booking_system_backend/server.py:190) and update `.env`.
- Frontend 5173: Vite will automatically try the next available port and print the actual URL.

### Backend crashes with `ModuleNotFoundError`
Your virtual environment is not activated. Run `source .venv/bin/activate` (macOS/Linux) or `.\.venv\Scripts\Activate.ps1` (Windows) from inside `booking_system_backend/`.

### `npm install` or `npm run dev` fails
Delete the `node_modules` folder and try again:
```bash
rm -rf node_modules
npm install
```

### Frontend shows "Failed to load flights"
The backend is not running. Start it first with `python server.py` from `booking_system_backend/`.

### `PowerShell execution policy` error on Windows
Run this once in PowerShell as Administrator:
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

# Galaxium Travels — AI Onboarding Pipeline (IBM Bob Hackathon Submission)

Built by Aman for the IBM Bob 2.0 Hackathon.

## The problem

Every time a new developer joins a project, they lose hours (sometimes days) just
figuring out how the codebase actually works before they can make a useful
contribution. They trace logic across files, hit undocumented quirks the hard way,
and end up interrupting teammates with basic questions — because docs are usually
missing, outdated, or were never written in the first place.

This repo (a fork of IBM's Galaxium Travels demo app) became my test bed for
fixing that, using IBM Bob 2.0.

## What I built

An onboarding pipeline inside Bob that takes this codebase and automatically
generates everything a new dev actually needs to get moving:

- **`AGENTS.md`** — generated with Bob's `/init`, giving persistent project context
  (stack, build/test/lint commands, architecture, and a few real "gotchas" like how
  dates are stored as ISO strings instead of proper datetime columns)
- **Architecture + sequence diagrams** (`docs/diagrams/`) — Mermaid diagrams
  showing the system's dual REST + MCP backend design, and a full sequence diagram
  tracing the `book_flight` flow including every error branch
- **A custom "Onboarding Guide" mode** (my own addition, not from any tutorial) —
  a reusable, global Bob mode that, for any project, generates:
  - `WALKTHROUGH.md` — a plain-English tour of how the app works
  - `SETUP.md` — a step-by-step local setup guide
  - `FIRST_TASKS.md` — concrete, ranked good-first-tasks with exact file/line
    references
- **Subagent delegation** — the first-tasks generation is handed off to a Bob
  subagent that independently re-scans the codebase in its own isolated context,
  rather than reusing whatever the main task already had loaded. This actually
  produced better, more concrete tasks than the first pass.

## Why this matters (impact)

I tested this by timing myself on a real task: understanding how `book_flight`
works, once by reading the raw source cold, once by reading the generated
walkthrough.

Reading `services/booking.py` directly took about **1 minute**. Reading the
generated `WALKTHROUGH.md` took about **1 minute 40 seconds**.

Honestly? For one small, well-written function, raw code was just as fast — there's
no hidden complexity to unpack. That's a real result and I'm not going to pretend
otherwise. But it pointed me to where the actual value is: **this app splits logic
across a REST layer, an MCP layer, a service layer, and a frontend that all have to
agree with each other**. Nobody figures that out by reading one file. The generated
docs surface that whole picture — the dual-protocol design, the shared validation
rules, the "why" behind decisions like returning `ErrorResponse` objects instead of
raising exceptions — in one place, instead of forcing a new dev to reconstruct it
by jumping between five files and guessing.

That's the real time sink this project targets: not reading one function, but
building a mental model of how a whole system fits together.

## Bob features used

- Agent mode (multi-step reasoning + file generation)
- `/init` for persistent project context
- Custom global modes (the differentiator — reusable across any project)
- Subagents (isolated context for the first-tasks analysis)
- Document understanding (reading the existing codebase to generate accurate docs)

Session summaries proving this usage are in [`bob_sessions/`](bob_sessions/).

## Repo structure

docs/diagrams/ → architecture + sequence diagrams
docs/onboarding/ → WALKTHROUGH.md, SETUP.md, FIRST_TASKS.md
bob_sessions/ → Bob task session summary screenshots

## Try it yourself

See [`docs/onboarding/SETUP.md`](docs/onboarding/SETUP.md) for full local setup
instructions for this app.

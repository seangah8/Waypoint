# Project Blueprint — AI Vacation Planner

> **How to use this document.** This is a high-level blueprint of the decisions and reasoning for this project. It is *not* the implementation plan. Read it fully, then produce a detailed, file-level implementation plan from it — starting with the walking-skeleton vertical slice described under "Build sequence." Use `/plan` for each session before writing code. Treat the decisions and reasoning here as fixed unless a genuine technical conflict forces a change, in which case flag it rather than silently diverging.

---

## What we're building

An AI-assisted vacation planner. The user picks a city (or cities) and some preferences, and the app generates a day-by-day itinerary of **real** places, grouped by area and ordered sensibly within each day.

**v1 scope:** single owner per trip, who enters preferences on behalf of the group they're travelling with. No live multi-user collaborative editing. (See "Out of scope / stretch goals.")

---

## Tech stack

- **Frontend:** React, with Zustand (client state) + TanStack Query (server state)
- **Backend:** Node.js
- **Database:** PostgreSQL via TypeORM
- **External data:** Google Places API (real places, coordinates, hours, ratings)
- **LLM:** Claude (Sonnet 4.6) via the Anthropic API
- **Clustering:** deterministic, in plain code (no model call)

---

## Core decisions (with reasoning)

**1. Places come from Google Places, the LLM only curates.**
The backend fetches a real, verified list of places from Google Places. Claude works **only** from that fetched list — it never invents places.
*Why:* grounding the model in real data makes hallucinated/fake places structurally impossible, and gives a clean tool-use story.

**2. Day clustering is deterministic code, not the LLM.**
Grouping places into days by geographic proximity is done with a simple deterministic algorithm (distance-threshold bucketing or nearest-neighbor over lat/lng). Claude's role starts *after* clustering: within each day's bucket it selects which places make the cut based on preferences, orders them, respects opening hours, and writes the reasoning.
*Why:* spatial clustering is iterative numerical computation; an LLM simulating it over coordinates is unreliable, non-deterministic, and untestable. A few lines of code solve it correctly every time, for free. The principle for the whole AI pipeline: **deterministic where correctness matters, LLM where judgment and language matter.**

**3. No Redux. Zustand + TanStack Query, split by state type.**
Zustand holds shared *client* state (in-progress trip, active day, UI flags). TanStack Query holds *server* state (fetched places, generated plans, saved trips) — caching, loading, and error handling without boilerplate. Single-component state stays in `useState`.
*Why:* Redux's ceremony and nested-immutable-update pain were a stated concern. The client/server split is the real architectural insight — most Redux bloat is server data managed as if it were client state.

**4. PostgreSQL + TypeORM, minimal schema, one source of truth.**
Tables: `users`, `places` (real-world catalog from Google Places; `city` as a plain field, no separate `cities` table yet), `trips` (owner, dates, `preferences`), `trip_stops` (join: which place, which trip, which day, what order).
`trip_stops` is the **single source of truth** — the only store of the plan. The LLM returns structured output that maps directly onto `trip_stops` rows, so there is no separate copy of the raw response in the schema. (If raw output is ever needed for debugging, log it to a file/console at that time — it does not belong in the database.)
*Why:* the data is genuinely relational, and reusing a known stack keeps the week's learning budget for the genuinely new parts. Keeping a single store — rather than the rows *plus* a raw JSONB copy — is the cleanest way to prevent two versions of the same data drifting apart.

**5. Build as a vertical slice (walking skeleton), not layer by layer.**
Build a thin end-to-end path through the riskiest flow first, then widen outward in feature-sized slices that each touch DB + backend + frontend together.
*Why:* surfaces problems in the newest, riskiest part (the AI pipeline) on day one or two while there's time to react, and produces a demoable, feature-by-feature commit history.

---

## AI pipeline (the core flow)

```
user input (city + preferences)
        ↓
backend fetches candidate places from Google Places API   [real data]
        ↓
deterministic clustering groups places into N days by proximity   [plain code]
        ↓
for each day's cluster → Claude selects, orders, respects hours, writes reasoning   [LLM judgment]
        ↓
structured day-by-day plan → saved to trip_stops → rendered
```

---

## Build sequence (sessions)

**Session 0 — Setup / skeleton.** Scaffold frontend + backend, connect Postgres, generate and hand-edit `CLAUDE.md` with these decisions. Prove the toolchain: real data flowing DB → API → screen, even as a placeholder.

**Session 1 — Core vertical slice (highest risk first).** Input form (city + preferences) → backend fetches Google Places → deterministic clustering into days → Claude selects/orders/reasons within each day → return structured plan → render as a plain, unstyled list. The goal is proving the full pipeline works end to end inside the real app.

**Session 2 — Persistence.** Minimal auth (`users`), save/load trips via `trips` + `trip_stops` (the source of truth). Reload a saved trip.

**Session 3 — Map + export.** Show a day's places on a map; per-day "open in Google Maps" link for live navigation (one day fits under the 10-waypoint consumer-link cap).

**Session 4 — Polish.** Loading/error states, UI/UX refinement, edge cases.

*Throughout:* `/plan` before each session; deterministic clustering, AI pipeline, and the state split read closely enough to defend on a whiteboard; `/security-review` and `/code-review` before shipping; entries appended to `LEARNINGS.md`.

---

## Claude Code workflow

- `/init` → generate `CLAUDE.md`, hand-edit with the locked architecture above.
- `/plan` before any significant change, grounded in this blueprint.
- `/output-style explanatory` for routine work (Insights while building). For the load-bearing pieces — clustering algorithm, AI prompt/response contract, state split — slow down and understand the code well enough to rebuild it from scratch if asked.
- At least one custom **skill** (`.claude/skills/`, the current successor to `.claude/commands/`) for a real recurring task. Concrete one to build: a `/log-learning` skill that reads what was just done/discussed in the session and appends a dated entry to `LEARNINGS.md` in the two-category format — making the log a one-word habit instead of a chore, and satisfying the custom-skill requirement.
- `/security-review` (security pass: injection, XSS, exposed credentials) **and** `/code-review` (correctness bugs in the diff) before shipping — they check different things.

---

## Learnings log

`LEARNINGS.md` at the repo root: append-only, raw dated entries (not polished prose), written at the end of each session and right after solving anything confusing. Kept split into two sections: **domain/stack** learnings and **Claude Code usage** learnings. Use the `/log-learning` skill (see Claude Code workflow) to append entries quickly at the end of each session.

---

## Out of scope / stretch goals (if time remains)

- **Multi-user trips** — multiple people invited onto one trip, shared editing. Deliberately scoped out of v1; would be the natural next step.
- **In-app routing** — OpenRouteService for stop ordering (TSP optimization) and drawn walking/driving legs. v1 delegates all navigation to Google Maps via export.
- **City-level place caching** — normalize `city` into its own table and serve repeat searches from the DB instead of re-hitting Google Places, with a freshness check.

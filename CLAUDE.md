# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## What this project is

An AI-assisted vacation planner. The user picks a city and preferences; the app generates a day-by-day itinerary of **real** places, grouped by geographic proximity and ordered sensibly within each day.

v1 scope: single trip owner, no live multi-user collaboration.

---

## Monorepo layout

```
client/     Vite + React frontend
server/     Node.js backend (Express)
```

Run everything from the repo root. Each sub-project has its own `package.json`.

## Common commands

```bash
# Install all deps (run once after cloning)
npm install --prefix client && npm install --prefix server

# Dev (run in separate terminals)
npm run dev --prefix client      # Vite dev server
npm run dev --prefix server      # ts-node-dev or nodemon

# Tests
npm test --prefix server         # Jest (backend)
npm test --prefix client         # Vitest (frontend)

# Single test file
npm test --prefix server -- --testPathPattern=clustering

# Type-check
npm run typecheck --prefix client
npm run typecheck --prefix server
```

---

## Locked architecture decisions

These are fixed. Flag conflicts rather than silently diverging.

### 1. Places come from Google Places — the LLM only curates

The backend fetches a verified list of candidate places from the Google Places API. Claude works **only** from that list — it never invents places. This makes hallucinated venues structurally impossible.

### 2. Day clustering is deterministic code, not the LLM

Geographic grouping (places → days) is done with a plain distance-threshold or nearest-neighbor algorithm over lat/lng coordinates. Claude's role starts *after* clustering: within each day's bucket it selects which places make the cut, orders them, respects opening hours, and writes the reasoning.

**Principle: deterministic where correctness matters, LLM where judgment and language matter.**

### 3. State split: Zustand (client) + TanStack Query (server), no Redux

- **Zustand** — in-progress trip, active day, UI flags
- **TanStack Query** — fetched places, generated plans, saved trips (caching, loading, errors)
- **`useState`** — single-component local state

### 4. Schema: PostgreSQL via TypeORM, `trip_stops` is the single source of truth

Tables: `users`, `places` (Google Places catalog; `city` as a plain field), `trips` (owner, dates, `preferences` JSONB), `trip_stops` (which place, which trip, which day, what order).

`trip_stops` is the **only** store of the generated plan. The LLM returns structured output that maps directly onto `trip_stops` rows — no separate raw-response copy in the database. Log raw output to console/file if needed for debugging.

### 5. Build as vertical slices, not layers

Each session delivers a thin end-to-end path that touches DB + backend + frontend together. The AI pipeline (the highest-risk piece) is validated in Session 1.

---

## AI pipeline

```
user input (city + preferences)
        ↓
backend fetches candidate places from Google Places API
        ↓
deterministic clustering groups places into N days by proximity  [plain code]
        ↓
for each day's cluster → Claude selects, orders, respects hours, writes reasoning  [LLM]
        ↓
structured day-by-day plan → saved to trip_stops → rendered
```

Claude (Sonnet 4.6) is called via the Anthropic API. It receives a pre-clustered list per day and returns structured JSON that maps directly to `trip_stops` rows.

---

## Session build sequence

| Session | Goal |
|---------|------|
| 0 | Scaffold client + server, connect Postgres, prove toolchain (data flowing DB → API → screen) |
| 1 | Full vertical slice: input form → Google Places → clustering → Claude → render plan |
| 2 | Persistence: auth, save/load trips via `trips` + `trip_stops` |
| 3 | Map view + "open in Google Maps" day export |
| 4 | Loading/error states, UI polish, edge cases |

Always `/plan` before a session. `/code-review` before merging a slice. `/security-review` before final submission.

---

## Custom skills

| Skill | Purpose |
|-------|---------|
| `/log-learning` | Appends a dated entry to `LEARNINGS.md` (two sections: domain/stack + Claude Code usage) |
| `/test-ai-pipeline` | Re-runs the Places → cluster → LLM pipeline against a fixed test input |

---

## Out of scope for v1

- Multi-user / collaborative trips
- In-app routing (TSP optimization, drawn map legs)
- City-level place caching with a `cities` table

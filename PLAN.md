# PLAN.md — Save My Tokens!

Status: **planning complete, no application code yet.** Durable rules live in `AGENTS.md`.

Scope for now: a local tool that records token usage and cost for OpenAI-compatible
`POST /v1/chat/completions` traffic sent through its own gateway, shows that stored data in a local
dashboard, and enforces user budgets before forwarding.

---

## 1. Recommended architecture (solo developer, intermediate level, low budget)

**Single Node.js/TypeScript process, no infrastructure, no paid services.**

| Concern              | Choice                                                                     | Why                                                                                                                                                                                                    |
| -------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Runtime              | Node 22 LTS, ESM                                                           | Already installed; one runtime for gateway, CLI and dashboard server.                                                                                                                                  |
| Language             | TypeScript (`strict`)                                                      | Catches the mistakes that hurt most in money math and stream handling.                                                                                                                                 |
| Package manager      | pnpm                                                                       | Installed; fast, disk-efficient, strict dependency resolution.                                                                                                                                         |
| HTTP server          | Fastify (v5)                                                               | Streaming (`text/event-stream`) passthrough, hooks for pre-flight budget checks, built-in JSON parsing, good docs.                                                                                     |
| Database             | SQLite via `node:sqlite` (`DatabaseSync`)                                  | Verified working on this machine with **zero** dependencies and no C++ toolchain. Wrapped in a tiny `db/` adapter so swapping to `better-sqlite3` is a one-file change if the experimental API churns. |
| Migrations           | Hand-rolled numbered SQL migrations + `schema_migrations`                  | ~60 lines, no ORM, full control, readable schema history.                                                                                                                                              |
| Dashboard            | React + Vite + TypeScript, built to static files and served by the gateway | Familiar, well documented, no server-side rendering needed; one build output directory. Charts as plain SVG/CSS so no charting dependency is required.                                                 |
| CLI                  | `node:util` `parseArgs` in `bin/smt.js`                                    | `smt start`, `smt db path`, `smt budget` — no CLI framework.                                                                                                                                           |
| Tests                | Vitest + a mock upstream Fastify instance                                  | Fast, TS-native, easy streaming tests.                                                                                                                                                                 |
| Config               | `config.json` in the app-data dir + env overrides + CLI flags              | No secret manager, keys stay in env.                                                                                                                                                                   |
| Distribution (later) | `npm i -g save-my-tokens` / `pnpm dlx`                                     | Free, no installers to maintain.                                                                                                                                                                       |

**Deliberately rejected:** Postgres/Docker (needs a daemon), `libsql`/Turso (remote-first),
Prisma/Drizzle (extra layer over five tables), Next.js (build and runtime weight for a local
dashboard), native modules (no compiler on Windows here), any hosted analytics or logging service,
and any cloud sync.

### Process shape

```
smt start
  ├─ 127.0.0.1:8787  /v1/chat/completions   → budget pre-flight → forward upstream → capture usage
  ├─ 127.0.0.1:8787  /api/*                 → read-only JSON from SQLite (+ budget mutations)
  └─ 127.0.0.1:8787  /                      → dashboard static assets
```

One data directory: config, `usage.db` (WAL), logs. Gateway and dashboard share one SQLite
connection owner (single writer) and one in-memory budget-reservation registry.

### Data model sketch (M1 target)

- `requests` — `id`, `started_at`, `completed_at`, `provider`, `model`, `requested_model`,
  `stream`, `status`, `http_status`, `latency_ms`, `prompt_tokens`, `completion_tokens`,
  `total_tokens`, `usage_source` (`upstream` | `estimated` | `unknown`), `cost_usd`,
  `cost_source` (`computed` | `unknown`), `price_version`, `budget_decision`
  (`allowed` | `warned` | `blocked` | `bypassed`), `error_kind`. **No body columns.**
- `budgets` — `id`, `scope` (`global` | `model`), `model`, `period` (`daily` | `monthly`), `limit_usd`,
  `mode` (`warn` | `block`), `enabled`, `created_at`.
- `schema_migrations` — `version`, `applied_at`.
- Price table: versioned JSON file in the repo, overridable by the user's config file.

---

## 2. Milestones

Sequential, each one shippable and testable on its own. Sizes are relative, not calendar promises.

### M1 — Foundation and storage **(first milestone)**

Scaffold the repo (git init, `package.json`, TS/ESLint/Prettier/Vitest config, `.gitignore`),
config loading with precedence rules, app-data path resolution, `node:sqlite` adapter with WAL and
migration runner, `requests` + `budgets` + `schema_migrations` tables, and typed repository functions
(insert request, list/filter, aggregate totals by model/day).

- **Depends on:** nothing.
- **Acceptance criteria**
  - `pnpm typecheck`, `pnpm lint`, `pnpm test` pass from a clean checkout.
  - `smt db path` prints the resolved database path for the current OS; directory is created
    on demand with restrictive permissions.
  - Migrations apply to an empty file, are idempotent on re-run, and a failed migration rolls back.
  - Repository round-trips a request row (including `usage_source`/`cost_source`/`unknown` cases) and
    returns correct daily/model aggregates from a temporary database.
  - No network calls, no prompt/response columns, no dependency beyond `node:` builtins.
  - `README.md` has quickstart; `AGENTS.md` unchanged in substance.

### M2 — Pricing and cost engine

Versioned price table (USD per 1M input/output tokens), user overrides, pure cost functions,
estimate helpers (character-based token heuristic, clearly labelled), and rounding/storage rules.

- **Depends on:** M1 (schema, `price_version`).
- **Acceptance criteria:** exact-cost tests incl. zero-token and unknown-model cases; unknown model
  yields `cost_source = unknown` and null cost, never `$0`; estimates are always distinguishable
  from measured usage; changing the table does not alter historical records.

### M3 — Gateway proxy (core value)

`POST /v1/chat/completions` passthrough: streaming and non-streaming, upstream base URL + API key
config, usage extraction (incl. `stream_options.include_usage`), request record persistence,
redaction of secrets/bodies in errors, loopback binding, timeouts, and clean client disconnects.

- **Depends on:** M1, M2.
- **Acceptance criteria:** against a local mock upstream, both modes forward correctly and preserve
  the response body/stream byte-for-byte; a record per request with correct `usage_source`; no bodies
  or keys in the database, API responses, or logs (asserted by tests); connection failures return a
  clear error without writing a fabricated cost; existing official OpenAI SDK clients work unchanged
  by pointing `baseURL` at the gateway.

### M4 — Dashboard on real data

Read-only API (`/api/summary`, `/api/requests`, `/api/models`) plus a React dashboard: spend and
tokens for today / 7 / 30 days, per-model breakdown, exact-vs-estimated indicator, recent requests
table, honest empty state.

- **Depends on:** M1 (M2/M3 populate data).
- **Acceptance criteria:** with an empty database the UI shows an empty state, not zeros-as-data;
  with seeded _test_ data (fixtures, clearly separate from real data) the numbers match direct SQL
  assertions; aggregates are labelled exact/partly estimated; no client-side secrets; accessible
  keyboard/focus basics.

### M5 — Budgets and enforcement

Budget CRUD (CLI + UI), pre-flight estimate check before forwarding, `warn`/`block` modes,
per-model and global daily/monthly periods, reservation registry for in-flight requests,
post-response reconciliation, blocked-request records, and an explicit documented bypass.

- **Depends on:** M3 (interception point), M4 (UI), M2 (estimates).
- **Acceptance criteria:** a request that would exceed a `block` budget is rejected with a clear
  error and recorded as `blocked` without contacting upstream; `warn` forwards and records `warned`;
  concurrent requests cannot overshoot a `block` limit beyond the documented single-process
  reservation semantics; period boundaries use local time and are tested across DST; enforcement
  fails closed if the database is unreadable, and the bypass is logged.

### M6 — Hardening, docs and distribution

Opt-in gateway auth token (required for non-loopback binding), schema/version upgrade checks,
rate/size guards, structured redacted logging, end-to-end test of gateway → DB → dashboard,
documentation for supported integrations and known limitations, privacy statement, packaging for
global install, and a `docs/` FAQ that states scope honestly.

- **Depends on:** M1–M5.
- **Acceptance criteria:** documented threat model (local attacker, malicious upstream, log
  leakage); non-loopback start without a token refuses to boot; upgrade from an older database is
  covered by a test; a first-time user can follow the README to see their own real request in the
  dashboard; every "estimate" surface states it is an estimate.

---

## 3. Dependency graph

```
M1 ──> M2 ──┐
 └──────────┴──> M3 ──> M5 ──> M6
      └──────> M4 ──> M5
```

M1 is a hard prerequisite for everything. M2 and M4 can be developed in either order once M1 lands;
M3 needs M2; M5 needs M3 and M4.

## 4. Major technical risks

| Risk                                                                                          | Impact                                     | Mitigation                                                                                                                                                                                            |
| --------------------------------------------------------------------------------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `node:sqlite` is still experimental in Node 22                                                | API/behaviour changes on Node upgrades     | Isolate in one adapter module; pin `engines >=22`; test suite runs against the adapter so a `better-sqlite3` swap is verifiable; document the warning.                                                |
| Streaming responses often omit `usage`                                                        | Cost shown as real when it is guessed      | `include_usage` requested where supported; otherwise `usage_source = estimated` and the UI says so; never label estimates as actuals.                                                                 |
| Token estimation is approximate without a tokenizer                                           | Pre-flight budget checks can be wrong      | Document that pre-flight uses a labelled heuristic estimate plus `max_tokens` worst case; reconcile with actual usage after the response; no `tiktoken` dependency in M1–M3 (revisit only with data). |
| Budget races across concurrent requests                                                       | Overspending a `block` limit               | Single-writer DB + in-memory reservation registry; atomic check-and-reserve; document that enforcement is per-process and resets on restart.                                                          |
| Provider variance in OpenAI-compatible APIs (headers, usage shapes, `stream_options` support) | Broken forwarding or missing usage         | Thin provider layer + mock-upstream contract tests; tolerate unknown usage; never throw after the response has started.                                                                               |
| Accidental secret/prompt leakage via logs or error passthrough                                | Privacy breach, the product's core promise | Redaction helpers, no body logging by default, tests asserting logs/DB/API never contain marker strings from test payloads.                                                                           |
| Gateway cannot see traffic that bypasses it                                                   | Misleading user expectations               | Explicit UI/docs wording about supported integrations; no multi-provider or subscription claims.                                                                                                      |
| Cost table drift from real vendor pricing                                                     | Wrong numbers, silently                    | Versioned table with `price_version` snapshot per record; documented update procedure; unknown models cost `unknown`, not `$0`.                                                                       |
| Windows-specific paths, `EPERM` on DB files, WAL behaviour                                    | Startup failures on the dev machine        | Path resolution tested on Windows in M1; WAL + busy timeout; no long-held locks.                                                                                                                      |
| Scope creep (multi-provider, dashboards for subscriptions, hosted sync)                       | Never ships                                | `AGENTS.md` non-goals; one milestone at a time.                                                                                                                                                       |

## 5. Next development task

**M1 — Foundation and storage.** Initialize the repository (git), add the TypeScript/pnpm/Vitest
scaffold with `.gitignore`, implement config + app-data path resolution, the `node:sqlite` adapter
with WAL and the migration runner, the initial schema, and the request/budget repositories with
tests — verified with `pnpm typecheck`, `pnpm lint` and `pnpm test`. No gateway, no UI, no network.

# AGENTS.md — Save My Tokens!

Guidance for anyone (human or AI agent) contributing to this repository. Read this file before
making changes. It records durable product requirements, architecture principles, privacy rules,
and development conventions. Milestones live in `PLAN.md`.

---

## 1. What this product is

**Save My Tokens!** is a privacy-first, **local-first** developer tool that:

1. Tracks AI token usage and cost for requests that pass through **supported integrations**.
2. Shows a dashboard built from **actually stored local data**.
3. Enforces **user-configured budgets** before AI requests are forwarded.

It is a local tool with a local gateway, not a hosted service.

## 2. Product principles (non-negotiable)

- **Local by default.** Usage data is stored in a local SQLite database. No cloud account, no
  sync, no telemetry, no analytics, no phone-home on startup or on request.
- **Real data only.** The dashboard and every report read from the local database. Never render
  placeholder, sample, seeded, or fabricated metrics as if they were the user's usage. Empty state
  must look empty and say so.
- **Actual vs. estimated is an explicit distinction.** Token counts and costs are labelled with
  their provenance (`upstream` vs `estimated`). Never present an estimate as measured usage.
  Never quietly mix the two in a single unlabelled number — aggregates must state whether they are
  exact, partly estimated, or estimated.
- **No prompt or response storage by default.** Request/response bodies are processed in memory to
  forward a request and to extract usage metadata, then discarded. Opt-in retention is not in scope
  unless a future milestone explicitly adds it behind a documented, default-off setting.
- **Honest scope.** We track and control only traffic that routes through the local gateway for
  supported integrations (OpenAI-compatible Chat Completions). We do **not** claim universal
  control over AI coding tools, IDEs, vendors, or subscription plans, and we cannot see usage that
  happens outside the gateway. Never imply otherwise in code, docs, UI copy, or marketing text.
- **Budgets are enforcement, not advice.** A budget check happens _before_ a request is forwarded.
  Enforcement modes and their limitations (e.g. single-process, race windows) must be documented
  rather than hidden.
- **Boring, maintainable technology.** Prefer few dependencies, standard library features, simple
  data flow, and readable code over clever abstractions. This project is built by a solo developer
  with intermediate experience and a low budget.

## 3. Privacy and security rules

- Only metadata is persisted per request: timestamps, model name, token counts and their source,
  computed cost, status/HTTP status, latency, streaming flag, budget decision, and a coarse error
  kind. No messages, no completions, no tool arguments, no metadata blobs, no user identifiers.
- **Never log** request or response bodies, message content, API keys, `Authorization` headers,
  cookies, or full upstream error payloads (they can echo prompt text). Redact instead.
- Upstream API keys are read from the environment or the user's config file and are passed to the
  upstream provider only. Never persist keys in the database, never return them from an API, never
  include them in error messages or crash output.
- The gateway binds to `127.0.0.1` by default. Binding to a non-loopback address requires an
  explicit setting plus a gateway access token; that pairing must be enforced in code, not
  documented as a suggestion.
- The only outbound network calls allowed are the ones the user's request requires: forwarding to
  the configured upstream base URL. No update checks, no remote price lookups at runtime.
- Store the database and config in the OS user app-data directory (or a path the user explicitly
  sets). Create it with owner-only permissions where the platform supports it.
- Treat all incoming content (prompts, upstream error text, model names) as untrusted data. It must
  never reach a template, shell, SQL string concat, or log line unsanitized.

## 4. Architecture principles

- **One local process** exposes the gateway and the dashboard (separate routes, one binary/CLI).
  Avoid microservices, queues, containers, and background daemons.
- **Single writer to SQLite**, WAL mode, explicit transactions, append-only request records. Reads
  for the dashboard are plain queries — no cache, no derived metric store that can drift.
- **Plan the estimate/actual split into the data model**, not into the presentation layer. A record
  always carries its token/cost provenance.
- **Budget decisions are recorded** alongside the request record so enforcement is auditable.
- **Cost is computed at request time** and snapshotted onto the record together with the price
  table version used. Changing prices must never rewrite history.
- **Thin integration boundary.** The gateway owns a small, well-typed layer for "provider
  behaviour" (base URL, header handling, usage extraction from stream and non-stream responses)
  rather than scattering provider-specific `if` statements through request handling.
- **No silent failures.** If usage extraction or cost computation fails, persist the request with
  an explicit `unknown` provenance instead of inventing numbers.
- Keep domain logic (pricing, budget evaluation, aggregation) in pure functions with unit tests,
  independent of HTTP and SQLite.

## 5. Development conventions

- Language: **TypeScript**, ESM, `strict` mode; no `any` in public interfaces (use `unknown` +
  narrowing).
- Package manager: **pnpm** (lockfile committed). Node.js **22 LTS or newer** (see §7).
- Formatting/linting: Prettier + ESLint, non-interactive, run in CI and locally.
- Tests: **Vitest**. Unit tests for pure logic, integration tests against a temporary SQLite file,
  and gateway tests against a local mock upstream. No test may hit the real internet or need a real
  API key.
- Schema changes go through numbered, forward-only **migrations** with a `schema_migrations` table;
  never edit an applied migration.
- Config precedence: CLI flag > environment variable > config file > default. Document every
  setting where it is defined.
- Commits: small and focused, imperative subject line. Do not commit `.env` files, databases, build
  output, or keys. Do not add AI attribution footers unless the user asks.
- Docs: update `README.md` for user-facing behaviour and `PLAN.md` for milestone status. Any
  limitation we discover (e.g. "streaming from provider X returns no usage") belongs in the docs,
  not only in a code comment.
- Definition of done for a milestone: acceptance criteria in `PLAN.md` met, tests passing,
  `pnpm typecheck` and `pnpm lint` clean, docs updated, and no new dependency that could have been
  the standard library.

## 6. Explicit non-goals (do not build without a milestone that says so)

- TLS interception, proxy auto-configuration, or anything that captures traffic not explicitly sent
  to the gateway.
- Scraping vendor dashboards, reading other apps' credentials, or estimating subscription usage.
- Multi-user accounts, hosted sync, team billing, or remote hosting.
- Storing prompt/response content, embeddings, or file attachments.
- Support claims for providers that are not OpenAI-compatible Chat Completions.

## 7. Verified local environment (checked at project start)

| Item                   | Value                                                              |
| ---------------------- | ------------------------------------------------------------------ |
| OS                     | Windows 11 (MINGW64 / MSYS bash)                                   |
| Node.js                | v22.16.0                                                           |
| npm / pnpm / yarn      | 10.9.2 / 10.32.1 / 1.22.22                                         |
| Bun                    | not installed                                                      |
| Python                 | 3.13.3                                                             |
| Git                    | 2.56.0 (repository was **not** initialised at project start)       |
| SQLite CLI             | 3.44.0                                                             |
| MSVC / C++ build tools | **not** present — avoid native modules that need compilation       |
| `node:sqlite`          | works on Node 22.16 without a flag; emits an `ExperimentalWarning` |

Consequences: use `node:sqlite` behind a small adapter (see `PLAN.md` §Architecture), keep
everything else pure JavaScript/TypeScript.

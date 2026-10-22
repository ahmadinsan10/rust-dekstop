# Building a Backend in Rust — A Hands-On Guide

This guide tells you **what to do**, step by step. It does not give you code to copy.
You write every line yourself.

**Stack:** [Axum](https://github.com/tokio-rs/axum) + [Tokio](https://tokio.rs) + [SQLx](https://github.com/launchbadge/sqlx) + PostgreSQL + Serde + tracing.
**End result:** a production-style REST API for a **Tasks app** with user accounts, JWT auth, a real database, tests, Docker, and CI.

---

## Table of Contents

0. [Prerequisites](#phase-0--prerequisites)
1. [Hello Server](#phase-1--hello-server)
2. [Routing & Handlers](#phase-2--routing--handlers)
3. [Shared State (In-Memory CRUD)](#phase-3--shared-state-in-memory-crud)
4. [Project Structure](#phase-4--project-structure)
5. [Error Handling](#phase-5--error-handling)
6. [Configuration](#phase-6--configuration)
7. [Database with SQLx](#phase-7--database-with-sqlx)
8. [Validation](#phase-8--validation)
9. [Authentication](#phase-9--authentication)
10. [Middleware & Observability](#phase-10--middleware--observability)
11. [Pagination, Filtering & Sorting](#phase-11--pagination-filtering--sorting)
12. [Testing](#phase-12--testing)
13. [Docker & Deployment](#phase-13--docker--deployment)
14. [Going Further](#phase-14--going-further)
- [API Specification](#api-specification)
- [Rust Concepts Checklist](#rust-concepts-checklist)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

---

## How to Use This Guide

- Do the phases **in order**; each builds on the last.
- After every step: `cargo run`, then hit the endpoint with `curl` / Bruno / Postman.
- Run `cargo clippy` and `cargo fmt` often.
- Keep docs.rs open for **your exact versions** (check `Cargo.lock`) — Axum's API changes between major versions.
- Each phase ends with a **✅ Checkpoint**. Commit to git at each one.
- Suggested location: a new folder `backend/` in this repo (`cargo new backend`).

---

## Phase 0 — Prerequisites

**Goal:** the background knowledge and tools you need.

1. **Rust basics** — ownership, borrowing, structs, enums, `match`, `Option`/`Result`, traits, modules. (The Rust Book ch. 1–10.)
2. **Async Rust** — work through the **Tokio tutorial** (tokio.rs/tokio/tutorial). Understand:
   - what a `Future` is and that it does nothing until `.await`ed,
   - why a runtime (Tokio) is needed,
   - `tokio::spawn`, and why shared data needs `Arc`,
   - `Send + 'static` requirements on spawned tasks.
3. **HTTP basics** — methods (GET/POST/PUT/PATCH/DELETE), status codes (200, 201, 204, 400, 401, 403, 404, 409, 422, 500), headers, JSON bodies.
4. **SQL basics** — `CREATE TABLE`, `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `WHERE`, `JOIN`, indexes, foreign keys.
5. **Tools to install:**
   - Docker Desktop (to run PostgreSQL),
   - an HTTP client: `curl`, **Bruno**, or Postman,
   - optional DB GUI: DBeaver, pgAdmin, or TablePlus.

✅ **Checkpoint:** you can explain what `.await` does and why `Arc` is needed to share data across tasks.

---

## Phase 1 — Hello Server

**Goal:** an HTTP server answering one route.

**Concepts:** async `main`, attribute macros.

1. `cargo new backend` and `cd backend`.
2. Add dependencies:
   - `cargo add axum`
   - `cargo add tokio --features full`
3. Make `main` async and annotate it with Tokio's main attribute macro.
4. Build an `axum::Router` with one route: `GET /health` → an async handler returning a static string `"ok"`.
5. Create a `tokio::net::TcpListener` bound to `0.0.0.0:3000`.
6. Call `axum::serve` with the listener and the router; `.await` it.
7. `cargo run`, then `curl http://localhost:3000/health`.

✅ **Checkpoint:** the response body is `ok`.

---

## Phase 2 — Routing & Handlers

**Goal:** understand how requests reach your code and how responses come back.

**Concepts:** extractors, `serde` derive, `impl IntoResponse`.

1. `cargo add serde --features derive` and `cargo add serde_json`.
2. **Extractors** — handler arguments that Axum fills from the request. Try each:
   - `Path` → route `GET /hello/{name}` returns `Hello, <name>` (path syntax differs between Axum versions — check docs: `{name}` in 0.8+, `:name` in older),
   - `Query` → `GET /add?a=1&b=2` deserializes into a struct with `a` and `b`, returns the sum,
   - `Json` → `POST /echo` takes a JSON body into a struct and returns it back as `Json`.
3. **Responses** — try returning:
   - a `String`,
   - `Json<T>` where `T: Serialize`,
   - a tuple `(StatusCode, Json<T>)` to set a status like `201 Created`,
   - `StatusCode::NO_CONTENT` alone.
4. Notice: the **order of extractors matters** — the one that consumes the body (`Json`) must be last.
5. Group routes: build a sub-router for `/api` and `nest` it into the main router.

✅ **Checkpoint:** all three extractor routes work via curl, including a `201` status.

---

## Phase 3 — Shared State (In-Memory CRUD)

**Goal:** full CRUD for tasks, stored in memory.

**Concepts:** `Arc`, `RwLock`/`Mutex`, `Clone`, `State` extractor, `uuid`.

1. `cargo add uuid --features v4,serde` and `cargo add chrono --features serde`.
2. Define a `Task` struct: `id` (Uuid), `title` (String), `description` (Option<String>), `done` (bool), `created_at` (DateTime<Utc>). Derive `Serialize`, `Deserialize`, `Clone`, `Debug`.
3. Define **separate input structs**: `CreateTask` (title, description) and `UpdateTask` (all fields `Option`, for partial updates). Never let clients set `id` or `created_at`.
4. Define an `AppState` struct holding the store: an `Arc<RwLock<HashMap<Uuid, Task>>>`. Derive `Clone` on `AppState` (cloning only clones the `Arc` pointer).
5. Attach state to the router with `.with_state(...)`; receive it in handlers with the `State` extractor.
6. Implement:
   - `GET /api/tasks` → list all,
   - `GET /api/tasks/{id}` → one, or `404`,
   - `POST /api/tasks` → create, return `201` + the task,
   - `PATCH /api/tasks/{id}` → apply only the provided fields,
   - `DELETE /api/tasks/{id}` → `204`, or `404`.
7. Keep lock guards short: lock, do the work, drop. **Never hold a `std` lock across an `.await`.**

✅ **Checkpoint:** full create → list → update → delete cycle works. Data disappears on restart (expected).

---

## Phase 4 — Project Structure

**Goal:** a layout that scales.

**Concepts:** modules, `pub(crate)`, separation of concerns.

Restructure into:

```
src/
  main.rs          ← startup only: config, logging, DB pool, router, serve
  app.rs           ← builds the Router (all routes + layers)
  state.rs         ← AppState
  config.rs        ← Config struct (Phase 6)
  error.rs         ← AppError (Phase 5)
  routes/
    mod.rs
    health.rs
    tasks.rs       ← HTTP handlers: parse input, call service, shape output
    auth.rs        ← (Phase 9)
  models/
    task.rs        ← Task, CreateTask, UpdateTask
    user.rs        ← (Phase 9)
  services/
    task_service.rs ← business logic
  repo/
    task_repo.rs   ← database queries only (Phase 7)
```

Rules:
1. **Handlers are thin**: extract → call service → return response.
2. **Services** hold business rules (e.g. "title can't be changed once done").
3. **Repos** only talk to the database.
4. Make `app.rs` expose a function that returns the `Router` given an `AppState` — you'll reuse it in tests (Phase 12).

✅ **Checkpoint:** same behavior as Phase 3, `main.rs` is short, `cargo clippy` is clean.

---

## Phase 5 — Error Handling

**Goal:** consistent JSON errors, no panics, `?` everywhere.

**Concepts:** custom error enums, `thiserror`, `From` conversions, `IntoResponse`.

1. `cargo add thiserror` and `cargo add anyhow`.
2. In `error.rs`, define `AppError` with variants such as: `NotFound`, `BadRequest(String)`, `Validation(...)`, `Unauthorized`, `Forbidden`, `Conflict(String)`, `Internal(anyhow::Error)`.
3. Implement `IntoResponse` for `AppError`: map each variant to a status code and a JSON body like `{ "error": { "code": "not_found", "message": "..." } }`.
4. For `Internal`, **log the real error** but return a generic message to the client (never leak internals).
5. Implement `From<sqlx::Error>` (later) and `From<anyhow::Error>` so `?` converts automatically. Use `#[from]` in thiserror.
6. Change every handler to return `Result<impl IntoResponse, AppError>`.
7. Also handle **extractor rejections**: a malformed JSON body should return your JSON error format, not Axum's default plain text (look up custom extractors / `WithRejection`, or wrap `Json` in your own extractor).

✅ **Checkpoint:** missing task → `404` JSON; malformed body → `400`/`422` JSON; nothing ever panics.

---

## Phase 6 — Configuration

**Goal:** no hard-coded ports, URLs, or secrets.

**Concepts:** environment variables, `FromStr`, fail-fast startup.

1. `cargo add dotenvy`.
2. Create `.env` (add it to `.gitignore`!) and `.env.example` (committed, with placeholder values). Variables: `APP_PORT`, `DATABASE_URL`, `JWT_SECRET`, `RUST_LOG`.
3. In `config.rs`, a `Config` struct with a constructor that reads env vars and returns a `Result` — **fail at startup** with a clear message if something is missing.
4. Load `.env` at the top of `main` (only in development is fine).
5. Put `Config` (or the parts handlers need) into `AppState`.

> Alternative: the `config` or `figment` crates for layered config (file + env).

✅ **Checkpoint:** changing `APP_PORT` in `.env` changes the port; removing `JWT_SECRET` stops startup with a clear error.

---

## Phase 7 — Database with SQLx

**Goal:** persistent storage in PostgreSQL.

**Concepts:** connection pools, migrations, compile-time-checked queries, `FromRow`.

1. Start Postgres in Docker: create a `docker-compose.yml` with a `postgres:16` service, a user/password/db, port `5432`, and a named volume. `docker compose up -d`.
2. Set `DATABASE_URL=postgres://user:pass@localhost:5432/tasks` in `.env`.
3. Add SQLx: `cargo add sqlx --features runtime-tokio,tls-rustls,postgres,macros,uuid,chrono,migrate`.
4. Install the CLI: `cargo install sqlx-cli --no-default-features --features rustls,postgres`.
5. `sqlx database create`.
6. `sqlx migrate add create_tasks` → edit the generated SQL file yourself:
   - `tasks` table: `id UUID PRIMARY KEY`, `title TEXT NOT NULL`, `description TEXT`, `done BOOLEAN NOT NULL DEFAULT false`, `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`.
7. `sqlx migrate run`. Inspect the table in your DB GUI.
8. At startup: create a `PgPool` with `PgPoolOptions` (set `max_connections`), and run migrations automatically with the `sqlx::migrate!` macro.
9. Replace the `HashMap` in `AppState` with the `PgPool` (it's already cheap to clone).
10. In `repo/task_repo.rs`, write functions for each operation using **`sqlx::query_as!`** — SQLx connects to your DB **at compile time** and checks your SQL and types. Try misspelling a column and watch it fail to compile.
11. Use `RETURNING *` on `INSERT`/`UPDATE` to get the row back in one query.
12. For `PATCH`, use SQL `COALESCE($2, title)` so `NULL` params keep the existing value.
13. Map "no row found" (`fetch_optional` returning `None`) to `AppError::NotFound`.
14. Add `From<sqlx::Error> for AppError`; map unique-constraint violations to `Conflict`.

✅ **Checkpoint:** data survives server restarts; a typo in SQL is a compile error.

> Stretch: wrap a multi-step operation in a **transaction** (`pool.begin()`, `tx.commit()`).

---

## Phase 8 — Validation

**Goal:** reject bad input before it reaches business logic.

1. `cargo add validator --features derive`.
2. Annotate `CreateTask` / `UpdateTask` fields: title length 1–200, description max length, etc.
3. Build a custom extractor `ValidatedJson<T>` that:
   - deserializes like `Json<T>`,
   - calls `.validate()`,
   - returns `AppError::Validation` with **per-field messages** on failure.
4. Return `422 Unprocessable Entity` with a body listing each failing field.
5. Trim whitespace from titles in the service layer; reject titles that are empty after trimming.

✅ **Checkpoint:** an empty title returns `422` with a field-level message.

---

## Phase 9 — Authentication

**Goal:** users can register, log in, and only see their own tasks.

**Concepts:** password hashing, JWT, custom extractors (`FromRequestParts`), `spawn_blocking`.

1. Migration: `users` table — `id UUID PK`, `email TEXT UNIQUE NOT NULL`, `password_hash TEXT NOT NULL`, `created_at`.
2. Migration: add `user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE` to `tasks`, plus an index on it.
3. `cargo add argon2` (password hashing) and `cargo add jsonwebtoken`.
4. **Register** — `POST /api/auth/register`:
   - validate email format and password length (≥ 8),
   - hash with **Argon2** (random salt). Hashing is CPU-heavy → run it in `tokio::task::spawn_blocking`,
   - insert; on duplicate email return `409 Conflict`,
   - **never** return the hash in any response.
5. **Login** — `POST /api/auth/login`:
   - look up by email, verify the hash (also in `spawn_blocking`),
   - on failure return the **same generic 401** whether the email or password was wrong,
   - on success issue a **JWT** with claims: `sub` (user id), `exp` (e.g. 15–60 min), `iat`. Sign with `JWT_SECRET`.
6. **Auth extractor** — create an `AuthUser` struct and implement `FromRequestParts` for it:
   - read the `Authorization: Bearer <token>` header,
   - decode & validate the JWT (signature + expiry),
   - produce `AuthUser { id }` or reject with `401`.
7. Add `AuthUser` as an argument to every task handler. Every task query now filters by `user_id`.
8. Accessing another user's task should return **404** (not 403) so you don't reveal it exists.
9. `GET /api/auth/me` → returns the current user's id and email.

✅ **Checkpoint:** no token → 401; user A cannot see or modify user B's tasks.

> Stretch: refresh tokens stored in a DB table, logout (revoke refresh token), password change.

---

## Phase 10 — Middleware & Observability

**Goal:** logs you can debug with, plus standard HTTP hygiene.

**Concepts:** Tower layers, `tracing` spans.

1. `cargo add tracing tracing-subscriber --features tracing-subscriber/env-filter` (or add separately).
2. Initialize a subscriber at startup that respects `RUST_LOG` (e.g. `backend=debug,tower_http=info`). Try both pretty and JSON output.
3. `cargo add tower-http --features trace,cors,timeout,compression-full,request-id` and `cargo add tower`.
4. Add layers to the router:
   - `TraceLayer` — logs every request/response with latency,
   - **request ID** layer — generates an `x-request-id` and includes it in log spans,
   - `CorsLayer` — allow only your frontend origin (configurable),
   - `TimeoutLayer` — e.g. 10 s,
   - `CompressionLayer`.
5. Add `#[tracing::instrument]` to service functions (skip sensitive args like passwords!).
6. Add a `GET /health/ready` that checks the DB with a `SELECT 1`; keep `/health` as a simple liveness check.
7. **Graceful shutdown**: pass a shutdown signal (Ctrl+C via `tokio::signal`) to `axum::serve(...).with_graceful_shutdown(...)`.

✅ **Checkpoint:** each request produces a log line with method, path, status, latency, and request id; Ctrl+C shuts down cleanly.

---

## Phase 11 — Pagination, Filtering & Sorting

**Goal:** the list endpoint scales to many rows.

1. Define a `ListTasksQuery` struct (for the `Query` extractor) with optional: `page`, `per_page` (default 20, max 100), `done` (bool filter), `q` (search in title), `sort` (`created_at` / `title`), `order` (`asc` / `desc`).
2. Validate and clamp values in the service layer.
3. Build the SQL safely:
   - use bind parameters for values,
   - **never** interpolate `sort`/`order` from user input directly — map them through an enum to fixed SQL fragments,
   - look at `sqlx::QueryBuilder` for dynamic `WHERE` clauses.
4. Return a wrapper: `{ "data": [...], "page": 1, "per_page": 20, "total": 134 }`.
5. Add an index that supports your common query (`user_id, created_at`).

✅ **Checkpoint:** `GET /api/tasks?done=false&sort=title&order=asc&page=2` works and is fast.

> Stretch: cursor-based pagination instead of page/offset.

---

## Phase 12 — Testing

**Goal:** confidence without manual curl-ing.

**Concepts:** `#[tokio::test]`, `#[sqlx::test]`, integration tests, `tower::ServiceExt`.

1. **Unit tests** in services for pure logic (validation rules, pagination clamping).
2. **Integration tests** in `tests/`:
   - Use the router-builder function from Phase 4.
   - Call it with `tower::ServiceExt::oneshot` (send a `Request`, get a `Response`) — no real network needed. Or try the `axum-test` crate.
3. **Database tests**: annotate tests with `#[sqlx::test]` — it creates a **fresh database per test**, runs migrations, and hands you a `PgPool`.
4. Write tests for:
   - register → login → create task → list → it's there,
   - unauthorized access → 401,
   - user isolation (A can't read B's task),
   - validation failures → 422,
   - pagination totals.
5. Create small test helpers (e.g. "create user and return token") to keep tests readable.
6. Run: `cargo test`, `cargo clippy -- -D warnings`, `cargo fmt --check`.

✅ **Checkpoint:** all tests green; you can refactor without fear.

---

## Phase 13 — Docker & Deployment

**Goal:** run anywhere, automatically checked.

1. **Offline SQLx** — compile-time checks need a DB; for CI/Docker, run `cargo sqlx prepare` to generate the `.sqlx/` folder and commit it. Builds then use `SQLX_OFFLINE=true`.
2. **Dockerfile (multi-stage)**:
   - stage 1: official `rust` image → copy source → `cargo build --release`,
   - (optional) use **cargo-chef** to cache dependency builds between Docker builds,
   - stage 2: small runtime image (`debian:bookworm-slim` or distroless) → copy only the binary → run as a non-root user → `EXPOSE` the port.
3. Extend `docker-compose.yml` with your API service depending on Postgres (use a healthcheck so the API waits for the DB).
4. `docker compose up --build` → hit the API.
5. **CI (GitHub Actions)** — workflow on push/PR:
   - checkout, install Rust (with cache — e.g. `Swatinem/rust-cache`),
   - `cargo fmt --check`, `cargo clippy -- -D warnings`,
   - start a Postgres **service container**, run `cargo test`,
   - build the Docker image.
6. **Deploy** — pick one: Fly.io, Railway, Render, Shuttle (Rust-native), or a VPS with Docker. Use a managed Postgres. Set secrets as environment variables, never in the image.
7. Release checklist: `RUST_LOG=info`, strong random `JWT_SECRET`, CORS limited to your real frontend, HTTPS (handled by the platform or a reverse proxy like Caddy/Nginx).

✅ **Checkpoint:** a push to `main` runs CI; your API is reachable on a public URL.

---

## Phase 14 — Going Further

Pick topics as you need them:

| Topic | Crates / approach |
|---|---|
| API docs (Swagger UI) | `utoipa` + `utoipa-swagger-ui` |
| Rate limiting | `tower_governor` |
| Caching / sessions | Redis via `redis` or `fred`; `deadpool-redis` |
| Background jobs | `tokio::spawn` + channels; or `apalis` job queue |
| Scheduled tasks | `tokio-cron-scheduler` |
| WebSockets / real-time | Axum's `ws` feature |
| File uploads | Axum `Multipart`; store on S3 with `aws-sdk-s3` |
| Sending email | `lettre` |
| gRPC services | `tonic` + `prost` |
| GraphQL | `async-graphql` |
| ORM instead of raw SQL | `SeaORM` or `Diesel` |
| Metrics | `metrics` + `metrics-exporter-prometheus` |
| Distributed tracing | `opentelemetry` + `tracing-opentelemetry` |
| Load testing | `oha`, `k6`, or `wrk` |

---

## API Specification

What you should have at the end:

| Method | Path | Auth | Success | Description |
|---|---|---|---|---|
| GET | `/health` | – | 200 | Liveness |
| GET | `/health/ready` | – | 200 / 503 | DB reachable |
| POST | `/api/auth/register` | – | 201 | Create user |
| POST | `/api/auth/login` | – | 200 | Returns JWT |
| GET | `/api/auth/me` | ✔ | 200 | Current user |
| GET | `/api/tasks` | ✔ | 200 | List (paginated, filterable) |
| POST | `/api/tasks` | ✔ | 201 | Create |
| GET | `/api/tasks/{id}` | ✔ | 200 / 404 | Get one |
| PATCH | `/api/tasks/{id}` | ✔ | 200 / 404 | Partial update |
| DELETE | `/api/tasks/{id}` | ✔ | 204 / 404 | Delete |

Error body format (all errors):
`{ "error": { "code": "validation_failed", "message": "...", "fields": { ... } } }`

---

## Rust Concepts Checklist

| Concept | Where |
|---|---|
| `async` / `.await`, Futures | Phases 0–1 |
| Attribute & derive macros | Phases 1–2 |
| Generics (`Json<T>`, `State<T>`) | Phase 2 |
| `Arc`, `RwLock`, `Clone` | Phase 3 |
| Modules & visibility | Phase 4 |
| Custom error types, `From`, `?` | Phase 5 |
| Trait implementation (`IntoResponse`) | Phase 5 |
| Fallible constructors | Phase 6 |
| Compile-time macros (`query_as!`) | Phase 7 |
| Custom extractors (`FromRequest[Parts]`) | Phases 8–9 |
| `spawn_blocking` for CPU work | Phase 9 |
| Tower `Service` / `Layer` | Phase 10 |
| Enums for safe dynamic input | Phase 11 |
| Async tests | Phase 12 |
| Release builds, feature flags | Phase 13 |

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| "the trait `Handler` is not implemented" | A handler argument isn't an extractor, the body extractor isn't last, or the returned type isn't `IntoResponse`. Add `axum::debug_handler` attribute (from `axum-macros`) for a much clearer error. |
| "future cannot be sent between threads safely" | You hold a non-`Send` value (e.g. a `std::sync::MutexGuard`, `Rc`) across an `.await`. Drop it before awaiting, or use `tokio::sync::Mutex`. |
| `query_as!` fails: "DATABASE_URL must be set" | Set it in `.env`, or use `SQLX_OFFLINE=true` with a committed `.sqlx/` folder. |
| `query_as!` type mismatch on nullable column | Column is nullable → field must be `Option<T>`; or override with `"col!"` / `"col?"` syntax. |
| Route returns 404 unexpectedly | Path syntax differs between Axum versions (`{id}` vs `:id`), or a nested router prefix is wrong. |
| Connection refused to Postgres | Container not running (`docker compose ps`), wrong port/host, or the API container should use the service name (e.g. `db`) instead of `localhost`. |
| Login is slow / server stalls under load | Argon2 running on the async runtime → move to `spawn_blocking`. |
| CORS errors in browser | `CorsLayer` missing, wrong origin, or missing allowed headers (`Authorization`, `Content-Type`). |
| Errors look like plain text, not JSON | Extractor rejections not mapped — see Phase 5 step 7. |

---

## Resources

- **Tokio tutorial** — <https://tokio.rs/tokio/tutorial>
- **Axum docs** — <https://docs.rs/axum>
- **Axum examples** (best reference) — <https://github.com/tokio-rs/axum/tree/main/examples>
- **SQLx** — <https://github.com/launchbadge/sqlx>
- **tower-http** — <https://docs.rs/tower-http>
- **Zero To Production in Rust** (Luca Palmieri) — a book that builds a production email-newsletter backend; excellent companion to this guide
- **Async Book** — <https://rust-lang.github.io/async-book/>
- **OWASP API Security Top 10** — <https://owasp.org/API-Security/>

---

## Alternative Frameworks

- **Actix Web** — mature, very fast, actor heritage; similar concepts to Axum.
- **Rocket** — very ergonomic, macro-heavy; great developer experience.
- **Poem** — Axum-like with built-in OpenAPI support.
- **Loco** — "Rails for Rust", batteries-included on top of Axum + SeaORM.

Phases 4–13 (structure, errors, config, DB, auth, observability, testing, deployment) apply to all of them.

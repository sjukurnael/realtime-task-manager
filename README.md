# Task Manager

[![CI](https://github.com/sjukurnael/realtime-task-manager/actions/workflows/ci.yml/badge.svg)](https://github.com/sjukurnael/realtime-task-manager/actions/workflows/ci.yml)

Collaborative task management system with near-real-time sync across clients.

- **Backend**: Go (`server/`) — chi router, Postgres-backed store (via `pgx`), WebSocket hub for live updates, an append-only event log for efficient sync
- **Frontend**: Vanilla React + Vite + TypeScript (`client/`)

## Contents

- [Quick start (Docker, one command)](#quick-start-docker-one-command)
- [Code architecture](#code-architecture)
- [Technology choices](#technology-choices)
- [Database](#database)
- [Sync: the event log](#sync-the-event-log)
- [AI task breakdown](#ai-task-breakdown)
- [Caching, rate limiting, and backpressure](#caching-rate-limiting-and-backpressure)
- [Testing](#testing)
- [How I'd scale it over time](#how-id-scale-it-over-time)
- [Tradeoffs](#tradeoffs)
- [Running locally (native dev)](#running-locally-native-dev)

## Quick start (Docker, one command)

With Docker Desktop running:

```sh
cp .env.example .env   # optional: add an Anthropic API key to enable the AI breakdown
make up                # builds + starts Postgres, migrations, Go server, frontend
make db-seed           # optional: load demo projects/tasks/comments
```

Then open <http://localhost:3000> — open it in two browser windows to see
the real-time sync and presence. To try the **AI task breakdown** (the
"✨ Break down with AI" button on any task), put an `ANTHROPIC_API_KEY`
in `.env` before `make up` — Docker Compose reads it automatically; see
[AI task breakdown](#ai-task-breakdown). Without a key everything else
works and that one button explains it's not configured. `make down` stops everything;
`make db-reset` wipes the database. No local Go or Node needed for this
path. For hacking on the code itself, see
[Running locally (native dev)](#running-locally-native-dev).

## Code architecture

### Repo layout

```
.
├── .github/workflows/ci.yml   # CI: backend, frontend, and e2e jobs on every push
├── docker-compose.yml     # db only (default) or full stack (--profile app)
├── Makefile                # up / down / test / test-unit / test-e2e / db-* targets
├── server/                 # Go backend
│   ├── Dockerfile           # multi-stage build for the full-stack compose profile
│   ├── main.go              # entrypoint: DB pool, router, route table
│   ├── db.go                 # pgxpool connection + startup health check
│   ├── models.go              # the wire contract: domain structs, patches, stats, WS envelope
│   ├── tygo.yaml               # config for generating the TS side of that contract (make gen-types)
│   ├── openapi.yaml             # OpenAPI 3 spec, embedded in the binary (see docs.go)
│   ├── docs.go                   # serves the spec + Swagger UI at /api/docs
│   ├── store.go                   # all Postgres reads/writes; the only file that writes SQL
│   ├── ai.go                       # Claude client for the AI task breakdown (prompt, schema, output validation)
│   ├── validation.go               # dependency cycle / cross-project / completion checks + batch index-graph checks
│   ├── events.go                    # event type constants, payload structs, recordEvent, ListEventsSince
│   ├── handlers.go                   # HTTP handlers: decode request -> call store -> broadcast -> respond
│   ├── ratelimit.go                   # per-caller token bucket on mutating requests
│   ├── store_test.go                   # integration tests: validation, event-log atomicity, concurrency
│   ├── hub.go                           # WebSocket registry, presence, event broadcast + subscribe-time replay
│   ├── migrations/0001_init.{up,down}.sql   # schema (golang-migrate)
│   └── seed/seed.sql                          # demo projects/tasks/comments
└── client/                 # React frontend (Vite + TypeScript)
    ├── Dockerfile, nginx.conf       # build + nginx image for the compose profile (proxies /api and /ws)
    ├── playwright.config.ts, e2e/   # Playwright e2e suite (see Testing)
    └── src/                         # components, hooks, api client — walked through in
                                     # "Frontend" below. src/generated/api.ts is GENERATED
                                     # from the Go structs (make gen-types); *.test.ts are unit tests
```

### Backend

`main.go` builds a Postgres connection pool (`db.go`), constructs a `Store`
(`store.go`) and a `Hub` (`hub.go`), and wires them into a chi router. Every
HTTP handler in `handlers.go` follows the same shape: decode the request
body, call one `Store` method, and — if that method returned any durable
events — broadcast them over the hub, then write the JSON response.

`store.go` is the only file that talks SQL. Every mutating method
(`CreateTask`, `UpdateTask`, `DeleteTask`, `CreateComment`, `DeleteComment`,
`UpdateProject`) runs entirely inside one Postgres transaction: it performs
the write, validates via `validation.go` where relevant (dependency cycles,
cross-project references, "can't complete while blocked"), and — through
`events.go`'s `recordEvent` — claims a sequence number and appends an
`events` row, all before committing. See
[Sync: the event log](#sync-the-event-log) for why.

`hub.go` tracks connected WebSocket clients and does three things: relays
presence ("who's viewing what," driven by the client's `viewing` messages),
broadcasts durable events to every connected client after their transaction
commits, and — when a client subscribes to a project — replays anything
that client missed directly to its connection first. Each connection has
its own buffered send queue drained by a dedicated writer goroutine (with
write deadlines and ping/pong keepalive) — see
[Caching, rate limiting, and backpressure](#caching-rate-limiting-and-backpressure).

### Frontend

`App.tsx` switches between two top-level views: `ProjectList` (the
dashboard) and `ProjectDetail` (a single project's board). Both fetch their
initial data over REST via `api.ts`.

`live.ts` opens exactly one WebSocket per browser tab (not one per
component) and exposes a small pub-sub — `useWsEvents.ts` wraps it in React
hooks (`useWsEvents` for raw messages, `usePresence`/`useAllProjectsPresence`
for roster state, `useConnectionStatus` for the "Live"/"Reconnecting…"
pill). `identity.ts` gives each tab a stable display name and client ID via
`sessionStorage`, so opening two tabs naturally simulates two collaborators.

`ProjectDetail.tsx` is where the real-time sync logic lives: it seeds a
`lastSeq` from the project snapshot, subscribes over WS with it, and from
then on applies incoming event payloads directly to local `tasks`/`project`
state instead of refetching — including gap detection and a catch-up fetch
if it ever misses one (see [Sync](#sync-the-event-log)). `TaskPanel.tsx`
does the same in miniature for a task's comments. `KanbanBoard`/`TaskCard`
handle drag-and-drop status changes (optimistic, with rollback on a
rejected request — e.g. moving a blocked task to Done).

### REST API reference

**Interactive docs: <http://localhost:3000/api/docs>** (Swagger UI over the
OpenAPI 3 spec at `/api/openapi.yaml` — `server/openapi.yaml`, embedded in
the binary, covering every schema, status code, and the ETag/rate-limit
behavior).

API routes are rooted at `/api` (`/health` and `/ws` sit at the server
root); all mutating routes accept an `X-Actor` header (see
[Sync](#sync-the-event-log)) and every response/request body is JSON.

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | liveness check |
| GET | `/ws` | WebSocket upgrade |
| GET | `/api/docs` | Swagger UI (spec: `/api/openapi.yaml`) |
| GET | `/projects/` | list projects |
| POST | `/projects/` | create project |
| GET | `/projects/stats` | per-project dashboard aggregates (task/done/blocked counts, assignees, last edited) in one SQL pass |
| GET | `/projects/{projectID}/` | get one project (includes `lastSeq`) |
| PATCH | `/projects/{projectID}/` | update name/description/metadata |
| DELETE | `/projects/{projectID}/` | delete a project and everything under it |
| GET | `/projects/{projectID}/tasks` | list a project's tasks (ETag'd by `lastSeq` — unchanged projects answer 304, see [caching](#caching-rate-limiting-and-backpressure)) |
| POST | `/projects/{projectID}/tasks` | create a task |
| GET | `/projects/{projectID}/events?after=&limit=` | catch-up: events since `after`, capped at `limit` (max 500) |
| GET | `/tasks/{taskID}/` | get one task |
| PATCH | `/tasks/{taskID}/` | update title/status/assignedTo/configuration/dependencies |
| DELETE | `/tasks/{taskID}/` | delete a task |
| POST | `/tasks/{taskID}/breakdown` | AI: suggest subtasks for a task (no mutation; 503 without `ANTHROPIC_API_KEY`, see [AI task breakdown](#ai-task-breakdown)) |
| POST | `/tasks/{taskID}/breakdown/apply` | create reviewed subtasks + dependencies atomically; parent becomes the finish line |
| GET | `/tasks/{taskID}/comments` | list a task's comments |
| POST | `/tasks/{taskID}/comments` | add a comment |
| DELETE | `/comments/{commentID}` | delete a comment |

### WebSocket protocol

Client → server:
- `{"type":"viewing","projectId","taskId","lastSeq"}` — sent whenever the
  user opens/changes a project or task; on an actual project switch this
  triggers subscribe-time replay (see [Sync](#sync-the-event-log))
- `{"type":"rename","name"}` — display name change

Server → client:
- `{"type":"event","projectId","seq","eventType","payload","actor"}` — a
  durable event (task/comment/project mutation), whether pushed live or
  replayed on subscribe
- `{"type":"presence.updated","projectId","presence":[...]}` — full viewer
  roster for a project, sent on any join/leave/move/rename
- `{"type":"project.created","projectId","resourceId"}` /
  `{"type":"project.deleted","projectId","resourceId"}` — thin
  notifications kept outside the durable event log (see
  [Sync](#sync-the-event-log) for why); clients on the dashboard refetch
  the project list on either

## Technology choices

- **Go + chi + pgx (backend).** The realtime core of this app is "hold
  thousands of cheap concurrent WebSocket connections and fan events out
  to them" — goroutines and `gorilla/websocket` make that almost free, and
  chi stays a thin route table rather than a framework the domain has to
  live inside. `pgx` is used directly (no ORM): every mutation is a
  hand-written transaction, and the event log's correctness depends on
  exactly what happens inside those transactions, so hiding them behind an
  abstraction would obscure the most important code in the repo.
- **Postgres (storage).** The spec rules out managed realtime DBs, but the
  deeper reason: the sync model needs a mutation and its event-log append
  to commit **atomically**, and relational transactions give that for
  free. JSONB covers the schemaless parts (`metadata`, `customFields`),
  real foreign keys cover dependency/comment integrity, and one `BIGINT`
  counter per project row provides the gapless sequence.
- **WebSockets (transport), not SSE or polling.** The protocol is
  bidirectional — clients push `viewing`/`rename` upstream for presence
  while events flow down — which SSE can't do over one connection, and
  polling would either waste round trips or add latency. Fallbacks aren't
  needed for this deployment target.
- **Vanilla React + Vite + TypeScript (frontend).** There's no SEO or
  server-rendering requirement — it's an app behind a login in any real
  deployment — so Next.js would add moving parts without buying anything.
  The interesting frontend logic (event application, gap detection) is
  plain TypeScript, typed against a contract **generated from the Go
  structs** — `server/models.go` and `server/events.go` are the single
  source of truth, and `make gen-types` (tygo) emits
  `client/src/generated/api.ts` from them, so the two sides can't silently
  drift. `types.ts` re-exports the generated contract and adds only what
  Go's type system can't express: the closed `TaskStatus` union and
  precise per-event payload shapes.
- **dnd-kit** for drag-and-drop: small, headless, and doesn't fight the
  optimistic-update flow.

## Database

Postgres 16, run via Docker, with schema migrations and seed data. Every
mutation claims a per-project sequence number (`projects.last_seq`) and
appends a row to the append-only `events` table — see
[Sync: the event log](#sync-the-event-log) below for how that's used.

**Prereqs:** Docker only (no local Postgres, no local `golang-migrate`).

```sh
make db-reset    # tear down, start postgres, run migrations, load seed data
```

Other targets:

```sh
make db-up       # start postgres and wait for it to be healthy
make db-migrate  # run migrations (via the migrate/migrate Docker image)
make db-seed     # load server/seed/seed.sql
make db-psql     # open a psql shell against the running db
```

**Schema** (`server/migrations/0001_init.up.sql`): `projects`, `tasks`,
`task_dependencies` (join table, not an array column, so dependency edges
have real FK/cascade integrity), `comments`, and `events`. Notes:

- `tasks.status` is `todo | in_progress | done` only — there's no stored
  `blocked` status. Whether a task is blocked is derived from its
  dependencies' statuses at read time (as the app already does), so it can
  never go stale relative to the tasks it depends on.
- Cross-project dependency prevention (a task and its dependency must share
  a project) is enforced in application code, not a DB constraint, since it
  needs a cross-row comparison a `CHECK` can't express.
- `updated_at` is maintained by application code (`server/store.go` sets it
  to `now()` on every update), not a DB trigger.

**Seed data** (`server/seed/seed.sql`): two projects with fixed UUIDs —
"Website Redesign" (8 tasks spanning todo/in_progress/done, a dependency
chain where `Deploy` depends on `Build API` and `Write tests`, and `Write
tests` depends on `Build API`, plus 5 comments from `alice`/`bob` on 2
tasks) and "Mobile App" (3 simple tasks, no dependencies). `last_seq` is 0
and `events` is empty for both — event history starts once the API exists.

## Sync: the event log

Every mutation (task create/update/delete, dependency change, comment
add/delete, project update) runs in one DB transaction that: claims the
next `seq` for that project (`UPDATE projects SET last_seq = last_seq + 1
... RETURNING last_seq` — the row lock this takes serializes concurrent
writers to the same project, which is intentional, not a bottleneck at
this scale), performs the actual mutation, and inserts one row into
`events`. Only after that transaction commits does the server broadcast
the event over WebSocket. If anything fails, the whole transaction rolls
back — the `seq` is never burned and no event is ever recorded without its
state change, so the log is always gapless and an event's existence always
means its mutation really happened. (`project.created`/`project.deleted`
are deliberately not part of this log — deleting a project would cascade-
delete its own event rows before any client could see them, and project-
list membership isn't something a client watching one project's internals
needs to replay anyway. They keep the old thin `{type, projectId}`
broadcast instead.)

**Payload shapes** (`server/events.go`): `task.created` carries the full
new task; `task.updated` carries only the changed scalar fields;
`task.dependencies_changed` is always its own event (a single PATCH
touching both scalar fields and dependencies emits two events, two
sequential `seq` numbers, one transaction); `task.deleted` carries the IDs
of other tasks whose dependency list just lost this one
(`removedFromDependenciesOf`), so clients patch those locally instead of
needing a separate event per affected task; `comment.added`/
`comment.deleted` and `project.updated` are similarly minimal. Every
mutating request is attributed via an `X-Actor` header (the user's display
name — see `client/src/identity.ts`), stored as the event's `actor`.

**Catching up.** `GET /api/projects/:id/events?after=<seq>&limit=<n>`
returns events with `seq > after`, ascending, capped at 500. A fresh
client seeds its `lastSeq` from the project snapshot's `lastSeq` field
(not from this endpoint — `after=0` would walk the entire history). Over
WebSocket, the `viewing` message now carries `lastSeq`; on an actual
project switch (not just moving focus between tasks in the same project)
the server replays everything the client missed directly to that
connection, in order, before it's otherwise eligible for live broadcasts.

**Client-side gap detection** (`client/src/ProjectDetail.tsx`): every
incoming event is checked against a locally-tracked `lastSeq`. Equal to
`lastSeq + 1` → apply directly and advance. Less than or equal → duplicate,
ignore. Greater → a gap: buffer the event, fetch the missing range from
the catch-up endpoint, apply it, then drain the buffer (which may itself
still be gapped if more arrived mid-fetch, in which case it fetches again).
Reconnecting after a dropped connection is handled explicitly: a
reconnected socket is a brand-new server-side connection with no viewing
state, so when the connection comes back up the client re-sends `viewing`
with its current `lastSeq` — the server replays exactly the offline gap
and the client rejoins the presence roster. Applying
a payload is idempotent (e.g. `task.created` dedupes by id before
appending), which is what makes it safe to reapply events that arrive both
via subscribe-time replay and an independent REST fetch.

Not implemented: a "client is way too far behind, just refetch a fresh
snapshot instead of replaying thousands of events" fallback — noted as a
TODO in `server/hub.go` (`replayTo`) and `client/src/ProjectDetail.tsx`
(`fillGap`), since ordinary reconnect gaps are small and `ListEventsSince`
caps a single query at 500 rows regardless.

## AI task breakdown

The one AI feature (extended challenge, option 4): every task has a
"✨ Break down with AI" button that asks Claude to decompose it into
subtasks with dependencies. It's built as **two phases** around the
existing sync machinery rather than beside it:

1. **Suggest** — `POST /api/tasks/:id/breakdown` calls Claude
   (`claude-opus-4-8` with adaptive thinking and a structured-output JSON
   schema, via the official Go SDK) with the task's title/description and
   the project's other task titles (so it doesn't propose duplicates). The
   model returns 3–8 subtasks whose ordering is expressed as `dependsOn`
   **indices into its own array** — the subtasks don't have IDs yet.
   Nothing is persisted; the model's output is validated server-side like
   any untrusted input (index range, self-references, cycles, count ≤ 10)
   before it's returned.
2. **Review + apply** — the client shows the suggestions in a modal where
   the user can deselect any of them (deselection remaps the surviving
   indices — `remapSelectedSuggestions` in `taskUtils.ts`, unit-tested).
   One click posts the kept batch to `POST /api/tasks/:id/breakdown/apply`,
   which runs **one transaction**: insert every subtask, map indices onto
   the new UUIDs and insert the dependency edges, and extend the original
   task's dependencies with the batch's *terminal* subtasks (the ones
   nothing else in the batch depends on) — the original becomes the
   **finish line**, blocked until the whole breakdown is done. The batch
   re-validates from scratch; the client could send anything.

The payoff of the two-phase design: the expensive, slow, fallible AI call
is a pure read the user can review, retry, or abandon — while the mutation
is deterministic, atomic, AI-free, and rides the **existing event log
unchanged**: the apply emits N × `task.created` + 1 ×
`task.dependencies_changed` with contiguous seqs, so every connected
client's board updates live through the same `applyEvent` path as any
manual edit, with zero new sync code. A failed batch burns no seqs.

**Configuration.** `cp .env.example .env` and set `ANTHROPIC_API_KEY`
there — Docker Compose reads `.env` automatically, so `make up` just
works (an exported env var also works, and native dev sets it in the
shell running `go run .`; `.env` is gitignored). Without a key
the app is fully functional and the suggest endpoint answers a clear
`503`, which the modal surfaces with a retry button — CI and the e2e
suite run keyless and exercise exactly that degraded path. The nginx
proxy's `/api/` read timeout is raised to 180s because the model call can
legitimately take a minute (the server caps it at 120s).

## Caching, rate limiting, and backpressure

**HTTP caching via the event log.** `last_seq` is bumped by every mutation
in a project, atomically, in the same transaction — which makes it a
version stamp for the whole project. The tasks-list endpoint exploits
that: it returns `ETag: "<lastSeq>"` with `Cache-Control: no-cache`, so
the browser revalidates on every fetch and an unchanged project answers a
bodyless **304** — the client transparently reuses its cached copy. A
large task list only crosses the wire when something actually changed,
with zero client-side code and no cache-invalidation machinery: the sync
design's sequence counter *is* the invalidation key. (The dashboard
doesn't even need that: `GET /api/projects/stats` computes its stat cards
in one SQL pass server-side, so the project list never downloads task
lists at all.)

**Rate limiting** (`server/ratelimit.go`). Mutating requests pass through
a per-caller token bucket (15/s, burst 30 — far above human click rates).
This matters here specifically because of two amplification effects: every
mutation takes the project row lock that serializes all writers to that
project, and every mutation fans out over WebSocket to all connected
clients — so one runaway client (a PATCH loop from a buggy effect, say)
would tax everyone. Over-limit requests get a 429, which the frontend
already surfaces through its normal error path. Reads are never throttled:
they're cheap, cacheable, and gap-filling depends on them.

**WebSocket backpressure** (`server/hub.go`). Each connection has a
bounded send queue drained by its own writer goroutine — the sole writer
to that socket, with a deadline on every write. Broadcasts enqueue and
never block: a client whose queue fills (a slow or stalled reader) is
disconnected rather than allowed to wedge the broadcast for everyone else.
Dropping it is safe *because of* the event log: on reconnect it re-sends
`viewing` with its `lastSeq` and the server replays exactly what it
missed. Dead connections are detected by ping/pong keepalive (30s pings,
60s read deadline), so ghost entries can't linger in presence rosters.

## Testing

Three tiers — unit, integration, end-to-end — all run by CI on every push
(see [CI](#ci) below):

```sh
make test        # integration: Go suite against a real Postgres
make test-unit   # unit: Vitest over the client's pure logic
make test-e2e    # e2e: Playwright against the full Docker stack (runs `make up` first)
```

The e2e tier needs client deps plus a one-time browser download
(`cd client && npm install && npx playwright install chromium`).

### Unit tests

`client/src/*.test.ts` (Vitest) cover the client's pure logic in
isolation: blocked-task derivation (including the fail-safe for dangling
dependency ids mid-sync), relative-timestamp formatting boundaries, and
the deterministic avatar identity helpers.

### Integration tests

Integration tests (`server/store_test.go`) run against a real Postgres —
the suite creates and migrates its own `taskman_test` database, so dev
data is never touched, and it skips loudly (without failing the build) if
Postgres isn't running. They cover the properties that live in the
database rather than in Go, which unit tests with mocks couldn't
meaningfully exercise:

- **Event-log atomicity:** every mutation emits its event(s) with
  contiguous seqs; a *failed* mutation (e.g. a rejected dependency) burns
  no seq and leaves no event row.
- **Concurrency:** 25 goroutines creating tasks in one project produce a
  strictly gapless 1..25 sequence — the row-lock serialization the whole
  sync model rests on.
- **Dependency validation:** self/missing/cross-project/cycle rejection,
  and "can't complete while a dependency is incomplete."
- **Cascades and cleanup:** deleting a task reports which other tasks
  lost a dependency; deleting a project takes its tasks, comments, and
  events with it.
- **Contract details:** metadata/JSONB roundtrips, minimal `task.updated`
  payloads, one-PATCH-two-events for scalar+dependency changes, catch-up
  pagination, and the stats aggregates.

### End-to-end tests

`client/e2e/` (Playwright) drives the real composed stack — nginx-served
client, Go server, Postgres — through a real browser, covering the seams
nothing else can: project/task/comment lifecycles through the actual UI,
dependency blocking (a blocked task's completion is rejected with the
error surfaced, then succeeds once the dependency is done), and
drag-and-drop persistence across a reload.

The realtime specs are the flagship: each runs **two independent browser
contexts** (separate sessions and identities, like two people on two
machines) and asserts the core promise of the system — a project, task,
status change, or comment made in one browser appears in the other with
no reload; presence avatars show who's viewing the board and the open
task. A third spec simulates a genuine network drop (severing the
WebSocket + offline emulation), has the second user keep working, and
asserts the reconnecting client catches up via seq replay — the sync
design's recovery path, exercised end to end.

Tests create uniquely-named projects and clean up via the API, so they
can run against a database that also holds dev/seed data.

### CI

`.github/workflows/ci.yml` runs three jobs on every push and PR:

- **backend** — `go vet` + the integration suite against a Postgres
  service container, plus a contract-drift check: regenerates the tygo
  types and fails if `client/src/generated/api.ts` doesn't match the Go
  structs.
- **frontend** — lint (oxlint), unit tests, and a full type-check +
  production build.
- **e2e** — builds and starts the same Docker images `make up` uses, then
  runs the Playwright suite against them; gated on the first two jobs.
  Server logs and the Playwright report are uploaded on failure.

## How I'd scale it over time

Roughly in the order the bottlenecks would appear:

1. **Multiple server instances.** The one piece of state the Go process
   holds is the WebSocket hub. Today a mutation is only broadcast by the
   instance that handled it, so the first scaling step is publishing
   committed events to a shared channel — Redis pub/sub, or Postgres
   `LISTEN/NOTIFY` to stay dependency-free — with every instance's hub
   subscribing and fanning out to its own connections. Nothing about the
   client protocol changes: the event log stays the source of truth, and
   pub/sub is allowed to be lossy because seq-gap detection already
   catches anything a client misses. Presence rosters would move to Redis
   with per-entry TTLs so a crashed instance's clients age out.
2. **Snapshot fallback for far-behind clients.** The TODO noted above: if
   `lastSeq` is thousands behind, tell the client to refetch the project
   snapshot instead of replaying the log. This also unlocks **event-log
   compaction** — events older than the oldest snapshot any client could
   need can be archived or dropped.
3. **Pagination.** Task lists and comment threads are currently fetched
   whole (though the ETag revalidation means an unchanged list is never
   re-shipped, and the dashboard already gets aggregates instead of
   tasks). At "2MB+ project" scale, cursor pagination on
   `(project_id, created_at, id)` for tasks and comments, with the board
   lazily loading per column. The event-log design is what makes this
   cheap to add: clients already apply deltas, so a paginated initial load
   doesn't change the sync path at all.
4. **Sharper backpressure and quotas.** The basics are in (bounded
   per-connection send queues with slow-consumer disconnect; per-caller
   token buckets on mutations — see
   [above](#caching-rate-limiting-and-backpressure)). The next steps are
   authenticated per-user quotas instead of per-IP, and rate limits on
   inbound WS messages (presence updates) as well as HTTP.
5. **Database growth.** The existing indexes cover the hot paths
   (`tasks(project_id)`, `events(project_id, seq)`). Beyond that: read
   replicas for the list/snapshot endpoints, then partitioning `events` by
   project hash if the log outgrows one table. The per-project `last_seq`
   row lock intentionally serializes writers *within* a project, but
   projects are independent — write throughput scales with the number of
   projects, so a hot global lock never emerges.

## Tradeoffs

- **Per-project write serialization.** Claiming a seq takes a row lock on
  the project, so concurrent writes to one project queue up. Deliberate:
  it's what makes the log gapless, and a human team's write rate to a
  single project is nowhere near the ceiling.
- **Last-write-wins at field granularity, no OT/CRDT.** Two people editing
  the same task description at once → last save wins. Field-level patches
  keep the blast radius small (editing the title never clobbers a
  concurrent status change), and for task metadata that's the right
  cost/benefit; a CRDT would only pay off for collaborative long-text
  editing.
- **Dashboard refetches instead of applying deltas.** `ProjectList`
  refetches on any task event, in contrast to the board, which is fully
  delta-driven. What it refetches is two small responses (the project
  list and the `/projects/stats` aggregates — never task lists), so the
  simplicity costs little. Comment threads similarly refetch the open
  thread on comment events; the payloads are tiny.
- **Model types are generated; route wiring is hand-maintained.** The
  structural contract (every struct, field name, and payload shape) is
  generated from the Go source via tygo — CI fails if the generated file
  drifts from the structs. The routes themselves live in two hand-written
  places: the fetch wrapper in `api.ts` and the OpenAPI spec behind
  `/api/docs`. Both are covered by tests (e2e exercises every endpoint
  the UI uses) but neither is machine-derived from the router; with more
  surface area or more clients I'd generate the spec from the chi routes
  (or the routes from the spec) so that layer is machine-checked too.
- **No auth.** Identity is a self-chosen display name per tab
  (`sessionStorage`) — the right scope for demonstrating sync, and the
  `X-Actor` header/actor column is where a real authenticated principal
  would slot in.
- **Events are retained forever.** No compaction until the snapshot
  fallback (above) exists to bound how far back a client can need.

## Running locally (native dev)

For hacking on the code with hot reload. Requires **Go 1.25+**, **Node.js
+ npm**, and **Docker** (for Postgres only — the
[one-command path](#quick-start-docker-one-command) needs just Docker).

### Backend

The backend reads and writes Postgres directly (via `pgx`), so the
database must be up first: `make db-up` from the repo root (or
`make db-reset` for a clean slate).

```sh
cd server
go run .
```

Serves the API and WebSocket endpoint on `:8080` (override with `PORT`).
Connects to Postgres via `DATABASE_URL` (defaults to
`postgres://app:app@localhost:5432/taskman?sslmode=disable`, matching
`docker-compose.yml`, so no extra config is needed for local dev). The
server fails fast on startup if it can't reach the database.

### Frontend

```sh
cd client
npm install
npm run dev
```

Vite dev server proxies `/api` and `/ws` — see `client/vite.config.ts`.
Serves the app at `http://localhost:5173`.

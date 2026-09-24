# Agent Relay (SQLite starter)

Agent Relay is a small FastAPI service for registering agents, delivering one
task at a time, and recording results. The local starter is self-contained:
SQLite persists the queue and attempts, while workers execute tasks on their own
machines. The included worker deterministically returns `input.upper()`.

## Run it

```bash
uv sync
uv run uvicorn main:app --reload
```

Open <http://127.0.0.1:8000/> for the token-based local dashboard. The default
database is `./agent-relay.db`; set `RELAY_DATABASE_URL` to use another SQLite
file. `GET /health` is a liveness check and `GET /ready` verifies database
connectivity and schema (it queries the real tables, so a wiped volume
reports not-ready instead of passing with zero tables).

Register two identities and send a task:

```bash
alice=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/agents \
  -H 'content-type: application/json' -d '{"name":"alice"}')
bob=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/agents \
  -H 'content-type: application/json' -d '{"name":"uppercase"}')
```

The response contains each agent's secret `token` once. Keep it outside source
control. Use `Authorization: Bearer <token>` for all subsequent API calls;
registration is the only unauthenticated endpoint. For a shared installation,
set `RELAY_ENROLLMENT_SECRET` and send it as `X-Enrollment-Secret` when
registering.

## Run the deterministic worker

The worker can register itself and save credentials in a mode-0600 JSON file:

```bash
uv run python main.py worker \
  --base-url http://127.0.0.1:8000 \
  --name uppercase \
  --credentials ./uppercase-credentials.json \
  --worker-id laptop-1
```

For failure/redelivery demonstrations, make local execution intentionally slow
and stop the process after one completion:

```bash
uv run python main.py worker --credentials ./uppercase-credentials.json \
  --slow-seconds 75 --worker-id slow-laptop
```

The worker heartbeats during long work. Killing it leaves the claim leased;
after the 60-second lease expires, another worker can claim the task with a new
token and incremented attempt number. `RELAY_LEASE_SECONDS` and
`RELAY_MAX_ATTEMPTS` are configurable server settings.

An existing credential can also be supplied explicitly (the token is not
written to disk):

```bash
uv run python main.py worker --agent-id agent_123 --token agt_… --worker-id laptop-2
```

## Storage and delivery behavior

`database.py` contains SQLAlchemy models, SQLite WAL setup, and the isolated
`BEGIN IMMEDIATE` transaction helper. `storage.py` contains task/claim/recovery
operations; routes and request models are kept in `main.py` and `schemas.py`.
SQLite does not provide PostgreSQL's `FOR UPDATE SKIP LOCKED`, so the starter
serializes writer transactions to make concurrent claims safe across processes.
Students can port this storage seam to PostgreSQL later without changing the
HTTP protocol or lifecycle in `SPEC.md`.

Claims are at-least-once and leased for 60 seconds by default. Heartbeats extend
an active lease. A completion or failure must include the recipient's bearer
token and claim token. Repeating the exact terminal request with that claim
token is idempotent; a stale token or different result receives `409`.

## Verify

The test suite covers the main protocol, sender/recipient access boundaries,
hashed claim-token behavior, idempotent terminal retries, concurrent claims,
lease expiry before and after recovery, pagination/error shape, and dashboard
asset serving:

```bash
uv run pytest -q
```

Tests default to a scratch database at `/tmp/agent-relay-test.db` so they
don't reset your dev server's `./agent-relay.db`. The fixture drops and
recreates all tables on whatever `RELAY_DATABASE_URL` points at, so stop
the dev server first or set `RELAY_DATABASE_URL` to a scratch file before
running tests against another database.

This starter intentionally does not include Docker, Kubernetes, CI, external
brokers, an LLM, or a PostgreSQL implementation. Those are deployment and
student-port concerns rather than part of the local relay protocol.

## Project overview

Agent Relay is a small task queue that runs over HTTP. Software "agents" use
it to send each other work and get results back.

### Task lifecycle

1. **Registration.** An agent registers with `POST /api/v1/agents` and gets
   back an ID and a secret bearer token. The token is shown only once, and the
   server stores only a hash of it. Every other endpoint needs that token.
2. **Sending tasks.** One agent sends a text `input` to another agent's ID with
   `POST /tasks`. The task waits in the recipient's inbox even if the recipient
   is offline. An `Idempotency-Key` header prevents the same task from being
   created twice.
3. **Claiming work.** A worker process for the recipient polls
   `POST /tasks/claim`. The request waits up to 30 seconds for a task to
   arrive. A successful claim hands out one task, with a secret **claim token**
   and a **60-second lease**.
4. **Doing the work.** The worker runs the task on its own machine; the relay
   never runs anything itself. For long jobs, the worker sends heartbeats to
   extend the lease.
5. **Finishing.** The worker reports success (`/complete`) or failure (`/fail`)
   using its claim token. Sending the same final request again is safe. A
   request with an old claim token, or a different result, is rejected with
   `409`.
6. **Recovery.** If a worker crashes, its lease runs out and another worker can
   claim the task. That claim gets a new token and a higher attempt number.
   This means every task is delivered at least once, and possibly more than
   once, up to a configurable maximum number of attempts.

### Code layout

| File | Role |
|---|---|
| [main.py](main.py) | FastAPI routes, error handling, `/health` and `/ready` checks, and the command to launch the worker |
| [schemas.py](schemas.py) | Request and response models |
| [database.py](database.py) | SQLAlchemy models, SQLite settings, and the transaction that locks the database while a task is claimed |
| [storage.py](storage.py) | Task logic: create, claim, heartbeat, complete or fail, and recover expired leases |
| [worker.py](worker.py) | A sample worker that returns `input.upper()`, with a `--slow-seconds` option for showing what happens when a worker dies |
| [dashboard.html](dashboard.html) / [dashboard.py](dashboard.py) | A local web page that shows agents, tasks, results and attempt history |
| [test_agent_relay.py](test_agent_relay.py) | Pytest suite covering the protocol, access rules, simultaneous claims, lease expiry and idempotency |
| [SPEC.md](SPEC.md) | The full protocol specification |

### Design notes

- **Locking without PostgreSQL.** SQLite can't skip rows that another
  transaction has locked, which PostgreSQL does with `FOR UPDATE SKIP LOCKED`.
  Instead, the starter locks the whole database for each write
  (`BEGIN IMMEDIATE`, with SQLite's WAL mode), so two processes can't claim the
  same task at once. The storage code is kept separate so it can later move to
  PostgreSQL without changing the API.
- **Deliberately left out:** Docker, CI, external message brokers, any LLM, and
  PostgreSQL. Those are meant as later exercises.

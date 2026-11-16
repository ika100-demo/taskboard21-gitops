---
plan_id: 001-persistent-tasks-in-postgres-with-a-web
spec: 001-persistent-tasks-in-postgres-with-a-web
feature: Persistent tasks in Postgres with a web task list
gitops_app: ika100-demo/taskboard21-gitops
status: completed
repos:
- id: taskboard21-gitops
  shape: gitops-app
  summary: Declare Postgres for dev, wire it into the Task API, run two API instances and give the website
    the API address
  acs:
  - AC-001.1
  - AC-001.2
  - AC-001.3
  order: before-merge
  gitops:
  - resource: postgres
  - uses: postgres
    component: taskboard21-api
  - replicas: 2
    component: taskboard21-api
    instance: dev
  - env:
      TASK_API_URL: http://taskboard21-api
    component: taskboard21-web
  depends_on: []
  done: true
- id: taskboard21-api
  shape: service-python
  summary: Keep tasks in Postgres so they survive restarts and are shared by every instance
  acs:
  - AC-001.1
  - AC-001.2
  - AC-001.5
  - AC-001.7
  criteria:
  - Given tasks A, B and C were created through POST /tasks, when the service is restarted against the
    same database, then GET /tasks returns 200 with A, B and C first-to-last in creation order, each with
    the same id, title and due_date that POST /tasks returned (system AC-001.1)
  - Given no task is stored in the database, when a client sends GET /tasks, then the response is 200
    with an empty JSON array, also right after a restart (system AC-001.1) (replaces AC-001.6)
  - Given two service instances use the same database, when a task is created through POST /tasks on one
    of them, then GET /tasks on either instance includes it at the end of the list, and no two tasks returned
    share an id (system AC-001.2)
  - 'Given the database is reachable, when a client sends POST /tasks with {"title": "Buy milk", "due_date":
    "2026-12-01"} or with only a title, then the response is 201 with id, title and due_date (null when
    omitted) as before, and the next GET /tasks returns that task with the same fields; invalid input
    still gets 422 naming the field and stores nothing (system AC-001.5)'
  - 'Given the database cannot be reached, when a client sends GET /tasks or POST /tasks, then the response
    is 503 with the JSON body {"detail": "Task storage is unavailable"} within 3 seconds and no task is
    stored (system AC-001.7)'
  - 'Given the database cannot be reached, when the readiness probe calls GET /ready, then it answers
    503 {"status": "not ready"}; when the database is reachable again, GET /ready answers 200 {"status":
    "ready"} and GET /tasks lists every task created before the outage (system AC-001.7)'
  arguments: |
    Connection comes only from DATABASE_URL (PG* also set) injected by the gitops entry; no in-memory store remains in the
    deployed service. The API owns the schema: create/migrate it idempotently at startup so two replicas starting together
    are safe. Creation order must hold across instances (a database-assigned sequence or timestamp, not process state).
    Ids stay server-assigned strings (uuid4 is fine). Tests need a real Postgres (e.g. a test container or a devbox service).
  depends_on: []
  done: true
- id: taskboard21-web
  shape: web-nextjs
  summary: Task board start page that lists tasks and adds a task through the Task API
  acs:
  - AC-001.3
  - AC-001.4
  - AC-001.5
  - AC-001.6
  - AC-001.8
  criteria:
  - Given the Task API's GET /tasks returns tasks A, B and C in that order, when a person opens the start
    page /, then it lists A, B and C in that order, each with its title and its due_date as YYYY-MM-DD,
    or "No due date" when due_date is null (system AC-001.3)
  - Given the Task API's GET /tasks returns an empty array, when a person opens the start page, then it
    shows the text "No tasks yet" and the add-task form with a title field and a date field (system AC-001.4)
  - Given the start page is open, when a person enters the title "Buy milk", optionally picks a due date,
    and submits, then the page sends POST /tasks with that title and due_date (null when none), and within
    2 seconds the returned task appears at the end of the list without a page reload and both fields are
    emptied (system AC-001.5)
  - Given the start page is open, when a person submits a title that is empty or only whitespace, then
    "Title is required" shows next to the title field; when the title is longer than 200 characters, then
    "Title must be at most 200 characters" shows there; in both cases the typed values stay and no POST
    /tasks is sent (system AC-001.6)
  - Given the Task API answers GET /tasks with anything but 200 and a JSON array, or does not answer within
    2 seconds, when a person opens the start page, then the page still loads with status 200 and shows
    "Tasks are unavailable right now. Please try again later." instead of the list, with the form below
    it (system AC-001.8)
  - Given the start page is open, when a submitted POST /tasks gets an answer other than 201 or 422, or
    none within 2 seconds, then "Tasks are unavailable right now. Please try again later." shows above
    the form, the typed title and due date stay in the form, and the list is unchanged (system AC-001.8)
  arguments: |
    The Task API base URL comes from the server-side env var TASK_API_URL (cluster-internal, never NEXT_PUBLIC_*): call the
    API only from the Next.js server (Server Component for the list, a Server Action or app/api route handler for the add),
    never from the browser. Add TASK_API_URL to docs/env-vars.md. Tests stub the Task API per the Contract.
  depends_on: []
  done: true
gitops_pin: []
---

## Decomposition

Three repos, no build dependency between the two components. The Task API keeps its contract (spec A1) and only swaps its storage; the website is a new client of that unchanged contract (A2), so both build in parallel against the `## Contract` below. The gitops entry goes first (`before-merge`) so that `DATABASE_URL` exists in dev before the Postgres-backed API image deploys. No new repo and no shared library are needed: the only shared type is the task JSON, which the contract states.

## taskboard21-gitops — Declare Postgres for dev, wire it into the Task API, run two API instances and give the website the API address

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.3 · **Depends on:** []

- `resource: postgres`: the platform's in-cluster Postgres (A9), declared for dev only (non-goal: no database in staging or prod; both components run only in dev today).
- `uses: postgres` gives `taskboard21-api` `DATABASE_URL` and `PG*` (AC-001.1, AC-001.2).
- `replicas: 2` for `taskboard21-api` in dev, so the two-instance condition of AC-001.2 holds in the environment where it is verified.
- `env TASK_API_URL=http://taskboard21-api` for `taskboard21-web`: the API's ClusterIP Service on port 80 in the same namespace (AC-001.3 and every web criterion depend on it).

## taskboard21-api — Keep tasks in Postgres so they survive restarts and are shared by every instance

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.5, AC-001.7 · **Depends on:** []

Replaces the process-local `TaskStore` behind `get_task_store()` (the seam its spec 001 design left for this) with a Postgres-backed store. Owns durability and ordering (AC-001.1), sharing across instances and unique ids (AC-001.2), the API half of the add flow (AC-001.5) and the outage behaviour: 503 plus failing readiness (AC-001.7). Its spec 001 criterion AC-001.6 ("since the service started") no longer holds once tasks persist, so it is replaced.

## taskboard21-web — Task board start page that lists tasks and adds a task through the Task API

**Shape:** `web-nextjs` · **Criteria:** AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.8 · **Depends on:** []

The start page `/` holds the list and the form (A3). Owns the display format (A5), client-side title validation with the exact messages (A4, A7), the add-without-reload flow (AC-001.5) and the unavailable state (AC-001.8). It talks to the Task API only, through the server side of the Next.js app, because `TASK_API_URL` is a cluster-internal address the browser cannot reach.

## Contract

Caller: `taskboard21-web` (server side). Callee: `taskboard21-api`, base URL from `TASK_API_URL` (dev: `http://taskboard21-api`). JSON bodies, `Content-Type: application/json`, no authentication (spec non-goal). The resource contract is unchanged from the Task API's spec 001.

**Task object**

```json
{"id": "<string, server-assigned, unique in the environment>", "title": "<string, as sent>", "due_date": "YYYY-MM-DD" | null}
```

**`GET /tasks`**

- `200` → JSON array of every task in creation order, oldest first, no limit; `[]` when none is stored.
- `503` → `{"detail": "Task storage is unavailable"}` when the database cannot be reached.

**`POST /tasks`** — request `{"title": string, "due_date"?: "YYYY-MM-DD" | null}`; unknown fields ignored.

- `201` → the created task object; it is the last element of the next `GET /tasks`.
- `422` → FastAPI validation body `{"detail": [{"loc": ["body", "<field>"], "msg": "...", "type": "...", "input": ...}]}`; the field is the last element of `loc`. Title rule: 1 to 200 characters, not only whitespace; `due_date` must be a real calendar date. Nothing is stored.
- `503` → `{"detail": "Task storage is unavailable"}`; nothing is stored.

**`GET /ready`** (probe, not called by the website) — `200 {"status": "ready"}` when the database answers, else `503 {"status": "not ready"}`. `GET /health` stays `200` during a database outage, so pods are not restarted for it.

**Website rendering rules** — `due_date` shown verbatim as `YYYY-MM-DD`, `null` shown as "No due date"; list order is the API's order; the website does not re-sort.

### Errors

- Error bodies are JSON with a `detail` key: a string for `503`, FastAPI's array of `{loc, msg, type, input}` for `422`.
- The website validates the title before sending (same rule as the API), so a `422` is not expected; if one arrives for `title`, it shows "Title is required" (empty or whitespace) or "Title must be at most 200 characters" (over 200) next to the title field and keeps the input; a `422` for any other field is treated like an unlisted response.
- Any status or body the contract does not list (including `5xx`, a non-JSON body, a connection error, and a `200` that is not a JSON array) is treated as "API unavailable": the website shows "Tasks are unavailable right now. Please try again later." (instead of the list on load, above the form on submit), keeps the form's input, and its own page still returns `200`. It logs one line with the status or error, never the title.
- No retries: `POST /tasks` is not idempotent, and a retried `GET` would push the page past its time budget.

### Timeouts

- Website → `GET /tasks` (page render): 2 s total; on timeout, the unavailable state above.
- Website → `POST /tasks` (submit): 2 s total; on timeout, the unavailable state above the form, input kept. The website does not know whether the task was stored; a reload shows the truth.
- Task API → Postgres: connect timeout 1 s and statement timeout 2 s per request; when either is hit or the connection fails, the request answers `503` (well within the 3 s of the API criterion). `GET /ready` checks the database with a 1 s budget.
- When the database is down, readiness fails and Kubernetes takes the API pods out of the Service; the website then gets connection errors instead of `503`, which it treats the same way. Once the database is back, readiness passes and the stored tasks are served again.

## gitops-app PR

- `/system:ship` applies the `taskboard21-gitops` entry first (resource `postgres`, `uses` on `taskboard21-api`, `replicas: 2` in dev, `TASK_API_URL` on `taskboard21-web`) in one PR, before either component merges.
- No pins: both components run only in dev, which tracks `main`, and the database is dev-only by the spec's non-goals; staging and prod are untouched.
- After Argo reconciles dev: the Postgres resource is ready; two `taskboard21-api` pods are ready with `DATABASE_URL` set and `/ready` 200; `taskboard21-web` has `TASK_API_URL`. Smoke: create a `smoke-<uuid>` task via `POST /tasks`, restart the API deployment, and `GET /tasks` still lists it (AC-001.1); `http://taskboard21-web.taskboard21-dev.localhost/` shows the list and adding a task works (AC-001.3, AC-001.5).

# Local Coding Agent App with Bun + Codex App Server

## Goal

Build a local-first web app that:

1. Accepts a coding task
2. Creates an isolated git worktree
3. Launches a local Codex session in that worktree
4. Streams progress and logs
5. Runs validation commands
6. Marks the task ready for review

This app is for a **single local user** and is a stepping stone toward a future production system.

---

## Core Decisions

Use:

- **Bun** for the full TypeScript app runtime
- **Next.js** or another Bun-compatible web framework for the UI/API
- **Bun SQLite** (`bun:sqlite`) for persistence
- **Codex app-server** as the coding runtime
- **Git worktrees** for isolated workspaces
- **Local filesystem** for artifacts

Do NOT use:

- `better-sqlite3`
- cloud workflow systems
- distributed workers
- multi-user auth
- container orchestration
- a custom agent loop for v1

The coding runtime should be **Codex**, not a homegrown planning/edit loop.

---

## Important Architecture Insight

The system has two separate layers:

### 1. Control plane
Your app owns:
- tasks
- worktree creation
- status tracking
- logs/events
- validation
- review UI

### 2. Execution plane
Codex owns:
- coding session
- repo inspection
- file edits
- command execution
- streamed agent events

Your app does **not** replace Codex.  
Your app orchestrates Codex.

---

## High-Level Flow

```text
User creates task
  -> task saved in SQLite
  -> worker picks up task
  -> create git worktree
  -> start Codex app-server session in that worktree
  -> stream agent events into DB
  -> run validation
  -> collect artifacts
  -> mark ready_for_review
```

---

## Codex Integration Strategy

Use the **Codex app-server** directly over JSON-RPC.

The Codex app-server supports:
- stdio transport by default
- websocket transport experimentally

For this app, prefer **stdio** because it is simpler and local.

Your Bun worker should:

1. spawn `codex app-server`
2. send JSON-RPC messages over stdin
3. read streamed JSON-RPC events from stdout
4. persist those events
5. stop the session when complete

Do not depend on a Python SDK from Bun.
If a TypeScript SDK is available and mature, it can be introduced later, but v1 should work directly against the protocol.

---

## Bun Runtime Requirements

Use Bun for:

- running the web server
- API routes
- background worker
- SQLite access
- process spawning for git and Codex
- file I/O

Use Bun’s built-in SQLite API rather than `better-sqlite3`.

---

## Tech Stack

- Bun
- TypeScript
- Next.js App Router or Hono
- `bun:sqlite`
- Git CLI
- Codex CLI / Codex app-server
- Local filesystem

Optional later:
- SSE for live log streaming
- GitHub integration
- AI summary step after run

---

## Database Schema

Use Bun SQLite.

### tasks
- id
- title
- prompt
- repo_path
- base_branch
- status (`queued`, `preparing`, `running`, `validating`, `ready_for_review`, `failed`)
- branch_name
- worktree_path
- codex_session_id
- summary
- created_at
- updated_at

### task_runs
- id
- task_id
- started_at
- finished_at
- status
- exit_code

### task_events
- id
- task_id
- source (`system`, `codex`, `validation`)
- level (`info`, `warn`, `error`)
- event_type
- payload_json
- created_at

### task_artifacts
- id
- task_id
- type (`summary`, `diff`, `changed_files`, `validation_output`, `stdout_log`)
- path_or_text
- created_at

---

## Folder Structure

```text
/apps/web
  /app
    /api
      /tasks
      /tasks/[id]
      /tasks/[id]/run
      /tasks/[id]/events
  /components
  /lib

/packages/core
  db.ts
  types.ts
  worktree.ts
  codex-client.ts
  validation.ts
  artifacts.ts

/apps/worker
  worker.ts
```

If you want to keep it simpler, this can all live in one repo/app initially.

---

## Worktree Management

Each task gets its own git worktree.

### Example command

```bash
git fetch origin
git worktree add /tmp/agent-task-123 -b agent/task-123 origin/main
```

Store:
- `branch_name`
- `worktree_path`

Inside the worktree, write a `TASK.md` file containing:
- task title
- task prompt
- acceptance criteria
- validation hints

---

## Codex Client Responsibilities

Create a small Bun module that wraps app-server JSON-RPC.

### Responsibilities
- spawn the Codex app-server process
- send JSON-RPC requests
- parse streamed responses/events
- expose simple functions like:
  - `startSession(worktreePath, prompt)`
  - `sendUserMessage(sessionId, message)`
  - `subscribeToEvents(sessionId, onEvent)`
  - `stopSession(sessionId)`

### Important
The rest of your app should **not** know JSON-RPC details.  
Hide them behind a `codex-client.ts` abstraction.

---

## Worker Responsibilities

The worker should poll for queued tasks and process one at a time initially.

### Processing flow

1. fetch next queued task
2. set status to `preparing`
3. create git worktree
4. write `TASK.md`
5. start Codex session with `cwd = worktree_path`
6. persist streamed Codex events to `task_events`
7. wait until Codex reports completion
8. set status to `validating`
9. run validation commands
10. collect artifacts
11. mark task `ready_for_review` or `failed`

---

## Validation Strategy

Keep validation narrow in v1.

Examples:
- `bun run lint`
- `bun test path/to/file.test.ts`
- `pnpm lint`
- `pnpm test -- path/to/file`

The validation command should be configurable per repo.

Store:
- stdout
- stderr
- exit code

---

## Artifact Collection

After Codex completes, collect:

### Changed files
```bash
git -C <worktree> status --short
```

### Diff stat
```bash
git -C <worktree> diff --stat
```

### Optional full diff
```bash
git -C <worktree> diff
```

### Validation output
Captured from the validation process

### Final summary
Prefer one of:
- a summary returned by Codex in the run
- or a final post-processing step that asks Codex to summarize what changed

---

## UI Requirements

Create a minimal local UI with:

### Task list page
- queued/running/review-ready tasks
- title
- status
- created time

### New task page
Fields:
- title
- prompt
- repo path
- base branch

### Task detail page
Show:
- status
- branch name
- worktree path
- live or polled event log
- changed files
- validation output
- summary

No auth required in v1.

---

## Event Streaming

For v1, polling is acceptable.

A simple approach:
- UI polls `/api/tasks/:id` and `/api/tasks/:id/events` every 2 seconds

Later, add SSE if needed.

Do not overbuild real-time streaming first.

---

## Safety Constraints

Enforce all of these:

- all git and validation commands must run inside the target repo/worktree
- reject any path outside the configured repo root
- add timeouts to all subprocesses
- truncate large outputs before storing/displaying
- do not expose arbitrary shell execution from the UI

Codex can operate in the worktree, but your app should still enforce the workspace boundary.

---

## Bun-Specific Implementation Notes

Use:
- Bun’s `spawn` / subprocess support for git and Codex
- Bun file APIs for reading/writing artifacts
- `bun:sqlite` for local persistence

Avoid dependencies that assume Node-native SQLite bindings.

---

## Codex Prompting Strategy

When starting a Codex session, give it:

1. task goal
2. path to `TASK.md`
3. instruction to read the task file first
4. instruction to make targeted changes
5. instruction to run the smallest relevant validation
6. instruction to summarize:
   - files changed
   - what changed
   - validation run
   - known follow-ups

Do not try to design a complex multi-agent prompt system for v1.

---

## MVP Scope

Build only this:

- create task
- run worker
- create worktree
- start Codex in that worktree
- save streamed events
- run validation
- show review page

That is enough.

---

## Explicitly Out of Scope for v1

- GitHub PR creation
- multiple concurrent workers
- browser previews
- sandbox VMs/containers
- approvals/workflow branching
- external tool integrations
- chat inside the task UI

---

## Success Criteria

The MVP is successful if:

1. a task can be created locally
2. a worktree is created automatically
3. Codex runs in that worktree
4. progress is visible in the UI
5. validation runs automatically
6. the task ends in `ready_for_review` with:
   - changed files
   - logs
   - summary

---

## Suggested Build Order

### Phase 1
- Bun app setup
- SQLite schema
- create/list task APIs
- basic UI

### Phase 2
- worktree manager
- local worker process
- status transitions

### Phase 3
- Codex app-server client over stdio JSON-RPC
- persist streamed events

### Phase 4
- validation runner
- artifact collection
- review page

### Phase 5
- cleanup controls
- retry task
- optional summaries/improvements

---

## Final Instruction

Do not over-abstract.  
Do not introduce complex framework patterns early.  
This app is a local control plane for Codex-driven coding tasks.

The main architectural boundary is:

- app orchestrates
- Codex executes
- SQLite records
- git worktree isolates

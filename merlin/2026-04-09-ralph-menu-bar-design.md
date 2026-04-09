

## Objective

Build a small Swift-based macOS menu bar app that lets Caleb register Ralph loop projects, inspect their current state, detect when the associated Codex worker has died, and stop or restart loop supervision. The system should replace the current shell watchdog functionality with Swift code while preserving the useful bits of the current contract and state model.

## Recommended shape

Implement this as a single Swift package / Xcode project with three targets:

1. **RalphCore** — shared domain logic and persistence
2. **ralph** CLI — terminal control surface for registration, inspection, and optionally lifecycle actions
3. **RalphBar** macOS menu bar app — primary UI and in-process supervisor for v1

This keeps the implementation in one language, avoids bash drift, and gives both GUI and CLI surfaces the same behavior.

## Explicit v1 decisions

- The source of truth for tracked projects is a **global user file** at `~/.config/ralph/projects.json`.
- The app is a **menu bar app** as the primary UX.
- **No daemon in v1.** The menu bar app supervises loops while it is running.
- If the app quits, **supervision stops**. This should be visible in the UI.
- Existing bash Ralph scripts are **deprecated and replaced by Swift implementations**, not wrapped indefinitely.
- The loop state model should include the **Codex PID** so the system can detect worker death and show stale/broken state.

## Product goals

The app should let Caleb:

- register a repo as a Ralph-managed project
- see all registered Ralph projects in one place
- see current loop state for each project
- see whether the last known Codex worker process still appears alive
- stop a running loop
- restart a loop
- copy the project path
- inspect recent logs / last worker message

The CLI should let scripts or terminal workflows:

- register and unregister projects
- list registered projects
- inspect current status as text or JSON
- optionally start/stop/restart loops using the same shared core logic where practical

## Non-goals for v1

- a background daemon or launch agent
- remote control across machines
- auto-discovery by crawling the filesystem
- supporting multiple different loop engines with different semantics
- full historical analytics beyond current state + recent logs
- making loops survive app exit

## Architecture

### 1. RalphCore

`RalphCore` owns all domain behavior.

#### Responsibilities

- registry persistence
- per-project metadata and validation
- loop state persistence and loading
- process spawning / stopping
- status derivation
- Codex PID capture and liveness checks
- log and last-message access
- JSON models shared by CLI and app

#### Core domain types

Suggested models:

```swift
struct RalphProject: Codable, Identifiable {
    var id: UUID
    var name: String
    var projectPath: String
    var createdAt: Date
    var updatedAt: Date
    var notes: String?
}

struct RalphRegistry: Codable {
    var version: Int
    var projects: [RalphProject]
}

struct RalphLoopState: Codable {
    var projectID: UUID
    var status: RalphLoopStatus
    var supervisorPID: Int32?
    var codexPID: Int32?
    var lastExitCode: Int32?
    var lastStatusMessage: String?
    var lastUpdatedAt: Date
    var lastStartedAt: Date?
    var lastCompletedAt: Date?
}

enum RalphLoopStatus: String, Codable {
    case idle
    case starting
    case running
    case continueRequested
    case complete
    case blocked
    case unsafe
    case userStopped
    case workerError
    case supervisorStopped
    case codexMissing
}
```

A separate derived projection should power the UI:

```swift
struct RalphProjectStatusViewModel {
    var project: RalphProject
    var loopStatus: RalphLoopStatus
    var supervisorIsAlive: Bool
    var codexPID: Int32?
    var codexIsAlive: Bool?
    var summaryLine: String
    var lastUpdatedAt: Date
}
```

The important split is:
- **persisted raw state** for recovery and inspection
- **derived interpreted state** for UI display

### 2. Registry location

Use:

- `~/.config/ralph/projects.json`

Recommended adjacent paths:

- `~/.config/ralph/` — registry root
- `~/.config/ralph/projects.json` — tracked projects

Per-project runtime state should remain **inside the project repo** under `.ralph/` so each repo remains inspectable and portable.

That means each project still gets a boring local state directory like:

```text
repo/.ralph/
  state.json
  supervisor.log
  last-message.txt
  last-exit-code.txt
```

This preserves a good property from the shell version: each repo carries its own operational breadcrumbs.

## Current bash implementation being deprecated

The current Ralph bash implementation that this design should replace is:

- `~/merlin/skills/ralph-loop/scripts/ralph-watchdog.sh`
- the repo-local runner pattern represented by `.ralph/run-worker.sh`

The important distinction is:

- the **watchdog/supervisor script is being deprecated**
- the **repo-local worker script is also a candidate for replacement in v1**, not something we should preserve by default

This spec now assumes the long-term goal is to replace the bash scripts entirely with Swift code and/or declarative Ralph config.

## Replacing the shell watchdog

The current shell watchdog has a good shape worth preserving:

- one supervisor loop
- one bounded worker pass at a time
- append-only log
- machine-readable terminal statuses
- explicit stop conditions

Reimplement those semantics in Swift rather than embedding bash.

### Current useful contract to preserve

The worker ends each pass with exactly one machine-readable terminal line:

- `WATCHDOG_STATUS: continue`
- `WATCHDOG_STATUS: complete`
- `WATCHDOG_STATUS: blocked`
- `WATCHDOG_STATUS: unsafe`
- `WATCHDOG_STATUS: user-stopped`

For v1, preserve those same terminal states.

### Extend the contract for Codex PID

The worker path should additionally expose the active Codex process ID. There are two reasonable ways to do this:

#### Option A — stdout token

The worker emits an additional machine-readable line such as:

- `WATCHDOG_CODEX_PID: 12345`

#### Option B — state file

The worker writes a small JSON file during launch, for example:

- `.ralph/worker-runtime.json`

with content like:

```json
{
  "codex_pid": 12345,
  "started_at": "2026-04-09T16:00:00Z"
}
```

### Recommendation

Use **Option B as the durable source of truth**, and optionally also emit the stdout token for logs/debuggability.

Why:
- stdout is easy to parse but ephemeral
- a runtime file is easier for the app and CLI to inspect later
- it avoids having “status” depend entirely on the latest log transcript

## Supervisor model in v1

The **menu bar app** is the supervisor.

For each running project:

1. app starts a bounded worker pass
2. app tracks the child process PID
3. app waits for completion
4. app parses terminal status + Codex runtime metadata
5. app updates `.ralph/state.json`
6. if status is `continue`, app sleeps for configured poll interval and launches another pass
7. if status is terminal, app stops supervision for that project

### Important behavior

If the app exits:
- current supervision is interrupted or stops being managed
- on next launch, the app should inspect persisted state and mark projects appropriately
- if a project was previously `running` but no supervising process exists, show it as **supervisor stopped** or **codex missing**, depending on what survives

This gives an honest v1 behavior without pretending there is a daemon.

## Process model

### Recommendation for v1

Use **one in-process supervisor object per running project** inside the app.

Each supervisor object owns:
- project metadata
- poll interval
- current worker task/process handle
- state transitions
- persistence updates

Suggested type:

```swift
final class RalphProjectSupervisor: ObservableObject {
    func start()
    func stop()
    func restart()
}
```

A higher-level coordinator manages all projects:

```swift
final class RalphSupervisorStore: ObservableObject {
    var projects: [RalphProjectStatusViewModel]
    func registerProject(at path: String)
    func startProject(id: UUID)
    func stopProject(id: UUID)
    func restartProject(id: UUID)
    func refresh()
}
```

## Worker invocation

Rather than preserving a repo-local bash runner forever, v1 should aim to replace it with a **repo-local declarative Ralph config** that `RalphCore` can interpret directly.

Suggested file:

- `repo/.ralph/config.json`

This lets the Swift implementation own worker launch semantics directly instead of delegating to shell scripts.

### Why replace `run-worker.sh`

Replacing the repo-local runner script buys us:

- less shell quoting and escaping nonsense
- one canonical process-launch implementation
- easier PID capture and status modeling
- easier restart behavior
- a future path to app-editable loop configuration
- fewer hidden repo-specific side effects

### Recommended v1 config model

Each Ralph repo should provide declarative loop configuration, for example:

```json
{
  "version": 1,
  "prompt_file": ".ralph_watchdog_prompt.txt",
  "poll_seconds": 5,
  "worker": {
    "kind": "codex",
    "args": ["exec", "--full-auto", "--skip-git-repo-check"],
    "prompt_mode": "file"
  }
}
```

Possible fields:

- prompt file path
- poll interval
- worker kind (`codex` for v1)
- argument list
- working directory override if ever needed
- environment overrides if truly necessary

### v1 recommendation

For v1, the Swift system should:

- replace `ralph-watchdog.sh`
- replace `.ralph/run-worker.sh`
- read repo-local Ralph config
- launch the worker directly
- own PID capture, status parsing, logging, and restart behavior

That makes Ralph a real Swift system instead of a GUI wrapped around legacy bash.

## State files

Recommended per-project `.ralph/` files in v1:

```text
.ralph/
  config.json
  state.json
  worker-runtime.json
  supervisor.log
  last-message.txt
  last-exit-code.txt
```

### `state.json`

Suggested structure:

```json
{
  "project_id": "uuid",
  "status": "running",
  "supervisor_pid": 54321,
  "codex_pid": 12345,
  "last_exit_code": 0,
  "last_status_message": "continue",
  "last_updated_at": "2026-04-09T16:10:00Z",
  "last_started_at": "2026-04-09T16:09:30Z",
  "last_completed_at": null
}
```

### `worker-runtime.json`

Suggested structure:

```json
{
  "codex_pid": 12345,
  "started_at": "2026-04-09T16:09:31Z",
  "runner_pid": 54322
}
```

The app should treat this as advisory runtime metadata, not sole truth.

## Liveness detection

The app should detect two different failure classes:

### 1. Supervisor missing

If persisted state says `running`, but the app is not currently supervising and no known supervisor PID exists/alive check fails:
- show **supervisor stopped**

### 2. Codex missing

If the loop is expected to be active, but a recorded `codex_pid` no longer exists:
- show **codex missing**
- make this visually distinct from normal stopped/completed states

This distinction matters because “the app stopped managing it” and “the worker died unexpectedly” are different operator problems.

### macOS implementation note

Liveness can be checked with standard process existence checks (`kill(pid, 0)` or `ProcessIdentifier`-backed equivalents). The shared core should isolate this behind a small protocol for testability.

## CLI design

The CLI should stay intentionally small and boring.

Suggested commands:

```text
ralph register <path>
ralph unregister <path-or-id>
ralph list
ralph status [<path-or-id>] [--json]
ralph start <path-or-id>
ralph stop <path-or-id>
ralph restart <path-or-id>
ralph logs <path-or-id>
ralph path <path-or-id>
```

### v1 minimum CLI scope

Must-have:
- `register`
- `unregister`
- `list`
- `status --json`

Nice-to-have in same phase if cheap:
- `start`
- `stop`
- `restart`
- `logs`

The app can use shared core directly rather than shelling out to the CLI, but the CLI should still exist as a first-class operator surface.

## Menu bar app UX

### Primary interaction model

Clicking the menu bar icon opens a menu/popover showing all registered Ralph projects.

Each project row should show:
- project name
- short path or repo folder name
- current status badge
- Codex liveness indicator when relevant
- quick actions

### Project row actions

Per project:
- **Start** or **Restart**
- **Stop**
- **Copy Path**
- **Reveal in Finder** (optional but likely cheap and useful)
- **Show Last Message** / **Show Logs**

### Status language

Recommended user-facing states:
- Running
- Starting
- Waiting for next pass
- Completed
- Blocked
- Unsafe
- Stopped by user
- Worker error
- Codex missing
- Supervision stopped

Avoid exposing raw watchdog token names unless in logs/details.

### App-level affordances

Menu should also include:
- **Register Project…**
- **Refresh**
- **Quit**

If the app is responsible for supervision, the quit affordance should communicate the consequence clearly, e.g.:
- `Quit RalphBar (stops supervision)`

## Registration flow

### CLI registration

`ralph register /path/to/repo`

Behavior:
- validate path exists
- validate repo contains expected Ralph structure, or create minimal `.ralph/` scaffolding if that is part of v1 scope
- add/update project record in `~/.config/ralph/projects.json`
- avoid duplicate registration by canonicalized path

### App registration

The app should provide a file/folder picker for repo selection, then perform the same validation through shared core.

### Path normalization

Use canonicalized absolute paths and resolve symlinks where practical so the same repo does not get registered twice through slightly different spellings.

## Restart behavior

Restart should mean:

1. if a supervisor/worker is active, stop it cleanly
2. clear transient runtime state as appropriate
3. launch a fresh bounded worker pass
4. resume normal supervision loop

The app should not need special-case restart logic beyond stop + start orchestration.

## Logging and last-message behavior

Preserve the current useful artifacts:

- append-only supervisor log
- last worker message snapshot
- last exit code snapshot

The menu bar UI should expose a compact summary inline and a more detailed view on demand.

For v1, a simple sheet/window/log popover is enough. No need for a full log viewer unless it falls out naturally.

## Failure modes and behavior

### Missing repo-local config

If `.ralph/config.json` is missing:
- status becomes `blocked` or `workerError`
- details explain what file is missing or invalid

### Missing prompt/spec inputs

If the repo-local runner exits because required inputs are absent:
- preserve last message
- surface as `blocked` if that is what the worker reports

### Invalid status token

If the worker exits without a valid terminal status line:
- mark `workerError`
- preserve the output for inspection

### App crash / quit during running loop

On relaunch:
- read persisted state
- if prior state was `running` or `starting`, reconcile with process checks
- likely downgrade to `supervisorStopped` or `codexMissing`

## Testing strategy

### RalphCore tests

Test:
- registry load/store
- duplicate path handling
- path normalization
- status token parsing
- runtime metadata parsing
- state transitions
- liveness derivation
- restart behavior
- reconciliation after app relaunch

### CLI tests

Test:
- command parsing
- JSON output shape
- registry mutation behavior
- human-readable status output

### App tests

Test at minimum:
- menu renders project rows correctly for derived states
- row actions invoke correct core behaviors
- copy-path action yields canonical path

## Recommended implementation phases

### Phase 1 — Core + registry + status model

Build:
- `RalphCore`
- registry persistence
- state models
- status parsing
- liveness checks
- `.ralph/state.json` read/write

### Phase 2 — CLI

Build:
- register/unregister/list/status
- JSON output
- enough lifecycle hooks for app parity where cheap

### Phase 3 — App supervision engine

Build:
- in-app project supervisors
- bounded pass launching
- stop/restart logic
- log and state persistence

### Phase 4 — Menu bar UI

Build:
- project list
- status badges
- start/stop/restart controls
- copy path
- details/log access

### Phase 5 — Codex death detection polish

Build:
- explicit Codex missing detection
- better operator-facing warnings
- relaunch reconciliation polish

## Key tradeoffs

### Why not a daemon in v1?

Pros of avoiding daemon now:
- lower complexity
- fewer lifecycle and launch-agent issues
- simpler debugging
- easier initial shipping

Cost:
- loops do not survive app exit
- app owns supervision lifecycle

This is acceptable for v1 if stated clearly.

### Why keep a repo-local runner?

Pros:
- environment setup stays per repo
- easier migration from current Ralph flow
- avoids baking harness-specific behavior into the app

Cost:
- one extra moving part per repo

Still the right trade for v1.

### Why a menu bar app first?

Pros:
- status-first workflow matches the problem
- quick access to many loops
- feels operational rather than document-centric

Cost:
- less room for large detail views

Mitigation:
- use sheets/windows for details when needed

## Open questions

These do not block the design, but should be answered during implementation planning:

1. Should `ralph register` create missing `.ralph/config.json` scaffolding automatically, or fail until the project has already been initialized?
2. Should the CLI fully support lifecycle operations in v1, or should lifecycle remain app-only initially?
3. Should the app launch at login, given that supervision only exists while it is running?
4. What exact worker config schema should `RalphCore` support in v1 without overfitting too early?
5. Should copy-path copy the repo path only, or also offer “copy relative path from system root” if different wording is intended?

## Recommendation summary

For v1, build a Swift menu bar app called something like `RalphBar`, backed by a shared `RalphCore` library and a small `ralph` CLI. Store the project registry globally at `~/.config/ralph/projects.json`, keep per-project runtime state in each repo’s `.ralph/` directory, replace the current Ralph bash scripts with Swift supervision plus repo-local declarative config, preserve the existing bounded-worker contract where it is still useful, and extend it with durable Codex PID metadata. Let the app supervise loops while open, explicitly accept that there is no daemon yet, and make supervisor loss vs Codex death visible as separate states.

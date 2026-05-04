# Symphony Service Specification

Status: Draft v1 (language-agnostic)

Purpose: Define a service that orchestrates coding agents to get project work done.

## Normative Language

The key words `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, `RECOMMENDED`, `MAY`, and
`OPTIONAL` in this document are to be interpreted as described in RFC 2119.

`Implementation-defined` means the behavior is part of the implementation contract, but this
specification does not prescribe one universal policy. Implementations MUST document the selected
behavior.

## 1. Problem Statement

Symphony is a long-running automation service that continuously reads work from an issue tracker,
creates an isolated workspace for each issue, and runs a coding agent session for that issue inside
the workspace. This specification uses Linear as the primary tracker example and Codex as the primary
agent example, but the contracts are designed for any tracker backend and any coding agent that can
run as a subprocess.

The service solves four operational problems:

- It turns issue execution into a repeatable daemon workflow instead of manual scripts.
- It isolates agent execution in per-issue workspaces so agent commands run only inside per-issue
  workspace directories.
- It keeps the workflow policy in-repo (`WORKFLOW.md`) so teams version the agent prompt and runtime
  settings with their code.
- It provides enough observability to operate and debug multiple concurrent agent runs.

Implementations are expected to document their trust and safety posture explicitly. This
specification does not require a single approval, sandbox, or operator-confirmation policy; some
implementations target trusted environments with a high-trust configuration, while others require
stricter approvals or sandboxing.

Important boundary:

- Symphony is a scheduler/runner and tracker reader.
- Ticket writes (state transitions, comments, PR links) are typically performed by the coding agent
  using tools available in the workflow/runtime environment.
- A successful run can end at a workflow-defined handoff state (for example `Human Review`), not
  necessarily `Done`.

## 2. Goals and Non-Goals

### 2.1 Goals

- Poll the issue tracker on a fixed cadence and dispatch work with bounded concurrency.
- Maintain a single authoritative orchestrator state for dispatch, retries, and reconciliation.
- Create deterministic per-issue workspaces and preserve them across runs.
- Stop active runs when issue state changes make them ineligible.
- Recover from transient failures with exponential backoff.
- Load runtime behavior from a repository-owned `WORKFLOW.md` contract.
- Expose operator-visible observability (at minimum structured logs).
- Support tracker/filesystem-driven restart recovery without requiring a persistent database; exact
  in-memory scheduler state is not restored.

### 2.2 Non-Goals

- Rich web UI or multi-tenant control plane.
- Prescribing a specific dashboard or terminal UI implementation.
- General-purpose workflow engine or distributed job scheduler.
- Built-in business logic for how to edit tickets, PRs, or comments. (That logic lives in the
  workflow prompt and agent tooling.)
- Mandating strong sandbox controls beyond what the coding agent and host OS provide.
- Mandating a single default approval, sandbox, or operator-confirmation posture for all
  implementations.

## 3. System Overview

### 3.1 Main Components

1. `Workflow Loader`
   - Reads `WORKFLOW.md`.
   - Parses YAML front matter and prompt body.
   - Returns `{config, prompt_template}`.

2. `Config Layer`
   - Exposes typed getters for workflow config values.
   - Applies defaults and environment variable indirection.
   - Performs validation used by the orchestrator before dispatch.

3. `Issue Tracker Client`
   - Fetches candidate issues in active states.
   - Fetches current states for specific issue IDs (reconciliation).
   - Fetches terminal-state issues during startup cleanup.
   - Normalizes tracker payloads into a stable issue model.

4. `Orchestrator`
   - Owns the poll tick.
   - Owns the in-memory runtime state.
   - Decides which issues to dispatch, retry, stop, or release.
   - Tracks session metrics and retry queue state.

5. `Workspace Manager`
   - Maps issue identifiers to workspace paths.
   - Ensures per-issue workspace directories exist.
   - Runs workspace lifecycle hooks.
   - Cleans workspaces for terminal issues.

6. `Agent Runner`
   - Creates workspace.
   - Builds prompt from issue + workflow template.
   - Launches the coding agent app-server client.
   - Streams agent updates back to the orchestrator.

7. `Status Surface` (OPTIONAL)
   - Presents human-readable runtime status (for example terminal output, dashboard, or other
     operator-facing view).

8. `Logging`
   - Emits structured runtime logs to one or more configured sinks.

### 3.2 Abstraction Levels

Symphony is easiest to port when kept in these layers:

1. `Policy Layer` (repo-defined)
   - `WORKFLOW.md` prompt body.
   - Team-specific rules for ticket handling, validation, and handoff.

2. `Configuration Layer` (typed getters)
   - Parses front matter into typed runtime settings.
   - Handles defaults, environment tokens, and path normalization.

3. `Coordination Layer` (orchestrator)
   - Polling loop, issue eligibility, concurrency, retries, reconciliation.

4. `Execution Layer` (workspace + agent subprocess)
   - Filesystem lifecycle, workspace preparation, coding-agent protocol.

5. `Integration Layer` (Linear adapter)
   - API calls and normalization for tracker data.

6. `Observability Layer` (logs + OPTIONAL status surface)
   - Operator visibility into orchestrator and agent behavior.

### 3.3 External Dependencies

- Issue tracker API (Linear for `tracker.kind: linear` in this specification version).
- Local filesystem for workspaces and logs.
- OPTIONAL workspace population tooling (for example Git CLI, if used).
- Coding-agent executable that supports the targeted Codex app-server mode.
- Host environment authentication for the issue tracker and coding agent.

## 4. Core Domain Model

### 4.1 Entities

#### 4.1.1 Issue

Normalized issue record used by orchestration, prompt rendering, and observability output.

Fields:

- `id` (string)
  - Stable tracker-internal ID.
- `identifier` (string)
  - Human-readable ticket key (example: `ABC-123`).
- `title` (string)
- `description` (string or null)
- `priority` (integer or null)
  - Lower numbers are higher priority in dispatch sorting.
- `state` (string)
  - Current tracker state name.
- `branch_name` (string or null)
  - Tracker-provided branch metadata if available.
- `url` (string or null)
- `labels` (list of strings)
  - Normalized to lowercase.
- `blocked_by` (list of blocker refs)
  - Each blocker ref contains:
    - `id` (string or null)
    - `identifier` (string or null)
    - `state` (string or null)
- `created_at` (timestamp or null)
- `updated_at` (timestamp or null)

#### 4.1.2 Workflow Definition

Parsed `WORKFLOW.md` payload:

- `config` (map)
  - YAML front matter root object.
- `prompt_template` (string)
  - Markdown body after front matter, trimmed.

#### 4.1.3 Service Config (Typed View)

Typed runtime values derived from `WorkflowDefinition.config` plus environment resolution.

Examples:

- poll interval
- workspace root
- active and terminal issue states
- concurrency limits
- coding-agent executable/args/timeouts
- workspace hooks

#### 4.1.4 Workspace

Filesystem workspace assigned to one issue identifier.

Fields (logical):

- `path` (absolute workspace path)
- `workspace_key` (sanitized issue identifier)
- `created_now` (boolean, used to gate `after_create` hook)

#### 4.1.5 Run Attempt

One execution attempt for one issue.

Fields (logical):

- `issue_id`
- `issue_identifier`
- `attempt` (integer or null, `null` for first run, `>=1` for retries/continuation)
- `workspace_path`
- `started_at`
- `status`
- `error` (OPTIONAL)

#### 4.1.6 Live Session (Agent Session Metadata)

State tracked while a coding-agent subprocess is running.

Fields:

- `session_id` (string, `<thread_id>-<turn_id>`)
- `thread_id` (string)
- `turn_id` (string)
- `codex_app_server_pid` (string or null)
- `last_codex_event` (string/enum or null)
- `last_codex_timestamp` (timestamp or null)
- `last_codex_message` (summarized payload)
- `codex_input_tokens` (integer)
- `codex_output_tokens` (integer)
- `codex_total_tokens` (integer)
- `last_reported_input_tokens` (integer)
- `last_reported_output_tokens` (integer)
- `last_reported_total_tokens` (integer)
- `turn_count` (integer)
  - Number of coding-agent turns started within the current worker lifetime.

#### 4.1.7 Retry Entry

Scheduled retry state for an issue.

Fields:

- `issue_id`
- `identifier` (best-effort human ID for status surfaces/logs)
- `attempt` (integer, 1-based for retry queue)
- `due_at_ms` (monotonic clock timestamp)
- `timer_handle` (runtime-specific timer reference)
- `error` (string or null)

#### 4.1.8 Orchestrator Runtime State

Single authoritative in-memory state owned by the orchestrator.

Fields:

- `poll_interval_ms` (current effective poll interval)
- `max_concurrent_agents` (current effective global concurrency limit)
- `running` (map `issue_id -> running entry`)
- `claimed` (set of issue IDs reserved/running/retrying)
- `retry_attempts` (map `issue_id -> RetryEntry`)
- `completed` (set of issue IDs; bookkeeping only, not dispatch gating)
- `codex_totals` (aggregate tokens + runtime seconds)
- `codex_rate_limits` (latest rate-limit snapshot from agent events)

### 4.2 Stable Identifiers and Normalization Rules

- `Issue ID`
  - Use for tracker lookups and internal map keys.
- `Issue Identifier`
  - Use for human-readable logs and workspace naming.
- `Workspace Key`
  - Derive from `issue.identifier` by replacing any character not in `[A-Za-z0-9._-]` with `_`.
  - Use the sanitized value for the workspace directory name.
- `Normalized Issue State`
  - Compare states after `lowercase`.
- `Session ID`
  - Compose from coding-agent `thread_id` and `turn_id` as `<thread_id>-<turn_id>`.

## 5. Workflow Specification (Repository Contract)

### 5.1 File Discovery and Path Resolution

Workflow file path precedence:

1. Explicit application/runtime setting (set by CLI startup path).
2. Default: `WORKFLOW.md` in the current process working directory.

Loader behavior:

- If the file cannot be read, return `missing_workflow_file` error.
- The workflow file is expected to be repository-owned and version-controlled.

#### 5.1.1 Multiple Workflow Files (Recommended Extension)

Implementations may support multiple workflow files (for example `WORKFLOW-dev.md`,
`WORKFLOW-pr-review.md`) to serve different boards or purposes from the same codebase.

When multiple workflows are used:

- Each workflow file should map to a separate orchestrator instance with its own tracker config,
  workspace root, and server port.
- A registry file (for example `workers.json`) may map deployment names to their workflow file,
  project identifier, port, and workspace root.
- A top-level runner script should launch all orchestrators in parallel.
- Each orchestrator instance should be independently conformant to this spec — multi-workflow
  coordination is out of scope.
- Per-workflow state isolation (for example persistent budget files) should use the workflow name as
  a namespace to prevent cross-contamination.

### 5.2 File Format

`WORKFLOW.md` is a Markdown file with OPTIONAL YAML front matter.

Design note:

- `WORKFLOW.md` SHOULD be self-contained enough to describe and run different workflows (prompt,
  runtime settings, hooks, and tracker selection/config) without requiring out-of-band
  service-specific configuration.

Parsing rules:

- If file starts with `---`, parse lines until the next `---` as YAML front matter.
- Remaining lines become the prompt body.
- If front matter is absent, treat the entire file as prompt body and use an empty config map.
- YAML front matter MUST decode to a map/object; non-map YAML is an error.
- Prompt body is trimmed before use.

Returned workflow object:

- `config`: front matter root object (not nested under a `config` key).
- `prompt_template`: trimmed Markdown body.

### 5.3 Front Matter Schema

Top-level keys:

- `tracker`
- `polling`
- `workspace`
- `hooks`
- `agent`
- `codex`

Unknown keys SHOULD be ignored for forward compatibility.

Note:

- The workflow front matter is extensible. Extensions MAY define additional top-level keys without
  changing the core schema above.
- Extensions SHOULD document their field schema, defaults, validation rules, and whether changes
  apply dynamically or require restart.

#### 5.3.1 `tracker` (object)

Fields:

- `kind` (string)
  - REQUIRED for dispatch.
  - Reference value: `linear`
  - Implementations MAY support additional tracker kinds (for example `github` for GitHub Projects
    v2). Each tracker kind defines its own required fields, endpoint defaults, and auth mechanism.
- `endpoint` (string)
  - Default for `tracker.kind == "linear"`: `https://api.linear.app/graphql`
  - Default for `tracker.kind == "github"`: `https://api.github.com/graphql`
- `api_key` (string)
  - MAY be a literal token or `$VAR_NAME`.
  - Canonical environment variable for `tracker.kind == "linear"`: `LINEAR_API_KEY`.
  - Canonical environment variable for `tracker.kind == "github"`: `GITHUB_TOKEN`.
  - If `$VAR_NAME` resolves to an empty string, treat the key as missing.
- `project_slug` (string)
  - REQUIRED for dispatch when `tracker.kind == "linear"` (maps to Linear `slugId`).
  - Semantics are tracker-kind-dependent (for example `owner/repo` format for GitHub).
- `project_number` (integer, OPTIONAL)
  - Alternative project identifier for trackers that use numeric board IDs (for example GitHub
    Projects v2 board number).
  - REQUIRED for dispatch when `tracker.kind == "github"`.
- `status_field` (string, OPTIONAL)
  - Name of the status/state field on the tracker board. REQUIRED when the tracker does not have a
    fixed state field name (for example GitHub Projects v2 single-select fields).
- `active_states` (list of strings)
  - Default: `Todo`, `In Progress`
- `terminal_states` (list of strings)
  - Default: `Closed`, `Cancelled`, `Canceled`, `Duplicate`, `Done`
- `working_state` (string, optional)
  - When set, must be a member of `active_states`.
  - On dispatch, the orchestrator should transition the issue to this state in the tracker.
  - Acts as a distributed lock: issues in `working_state` are excluded from candidate selection
    unless already claimed by the current orchestrator instance.
  - See Section 8.9 for startup recovery behavior.
- `done_state` (string, optional)
  - When set, must be a member of `terminal_states`.
  - Names the explicit terminal state to which a successful verdict closes an issue. Implementations
    that route verdicts to states (Section 5.3.7) should validate that any close-to-terminal mapping
    targets `done_state` rather than inferring "first terminal state".
- `paused_states` (list of strings, optional)
  - States where an issue is parked but eligible to revive (worker- or operator-driven). Excluded
    from candidate selection while parked. Distinct from `terminal_states` because an externally
    posted comment or operator action can return them to `active_states`.
- `backlog_states` (list of strings, optional)
  - Operator-only park states (deferred work). Never a valid `verdict_state_map` target — config
    validation should reject any verdict route into a backlog state. Only operator skills may move
    issues into or out of these states.

#### 5.3.2 `polling` (object)

Fields:

- `interval_ms` (integer)
  - Default: `30000`
  - Changes SHOULD be re-applied at runtime and affect future tick scheduling without restart.

#### 5.3.3 `workspace` (object)

Fields:

- `root` (path string or `$VAR`)
  - Default: `<system-temp>/symphony_workspaces`
  - `~` is expanded.
  - Relative paths are resolved relative to the directory containing `WORKFLOW.md`.
  - The effective workspace root is normalized to an absolute path before use.

#### 5.3.4 `hooks` (object)

Fields:

- `after_create` (multiline shell script string, OPTIONAL)
  - Runs only when a workspace directory is newly created.
  - Failure aborts workspace creation.
- `before_run` (multiline shell script string, OPTIONAL)
  - Runs before each agent attempt after workspace preparation and before launching the coding
    agent.
  - Failure aborts the current attempt.
- `after_run` (multiline shell script string, OPTIONAL)
  - Runs after each agent attempt (success, failure, timeout, or cancellation) once the workspace
    exists.
  - Failure is logged but ignored.
- `before_remove` (multiline shell script string, OPTIONAL)
  - Runs before workspace deletion if the directory exists.
  - Failure is logged but ignored; cleanup still proceeds.
- `timeout_ms` (integer, OPTIONAL)
  - Default: `60000`
  - Applies to all workspace hooks.
  - Invalid values fail configuration validation.
  - Changes SHOULD be re-applied at runtime for future hook executions.

#### 5.3.5 `agent` (object)

Fields:

- `max_concurrent_agents` (integer)
  - Default: `10`
  - Changes SHOULD be re-applied at runtime and affect subsequent dispatch decisions.
- `max_turns` (positive integer)
  - Default: `20`
  - Limits the number of coding-agent turns within one worker session.
  - Invalid values fail configuration validation.
- `max_retry_backoff_ms` (integer)
  - Default: `300000` (5 minutes)
  - Changes SHOULD be re-applied at runtime and affect future retry scheduling.
- `max_concurrent_agents_by_state` (map `state_name -> positive integer`)
  - Default: empty map.
  - State keys are normalized (`lowercase`) for lookup.
  - Invalid entries (non-positive or non-numeric) are ignored.

#### 5.3.6 `codex` (object)

Note: Implementations using a different coding agent (for example Claude Code CLI) should define an
equivalent config section (for example `claude`) with fields appropriate to their agent's integration
model. The fields below are Codex-specific; the logical contract is: command to launch, policy
settings, and timeout values.

Logical field mapping for common alternative agents:

| Codex field | Logical purpose | Claude Code equivalent |
|-------------|----------------|----------------------|
| `command` | Agent executable | `command` (default `claude`) |
| `approval_policy` | Permission model | `allowed_tools` (list of tool names) |
| `thread_sandbox` | Sandbox mode | N/A (CLI manages its own sandbox) |
| `turn_sandbox_policy` | Per-turn sandbox | N/A |
| `turn_timeout_ms` | Turn time limit | `turn_timeout_ms` |
| `read_timeout_ms` | Startup timeout | `read_timeout_ms` |
| `stall_timeout_ms` | Inactivity timeout | `stall_timeout_ms` |

Fields:

For Codex-owned config values such as `approval_policy`, `thread_sandbox`, and
`turn_sandbox_policy`, supported values are defined by the targeted Codex app-server version.
Implementors SHOULD treat them as pass-through Codex config values rather than relying on a
hand-maintained enum in this spec. To inspect the installed Codex schema, run
`codex app-server generate-json-schema --out <dir>` and inspect the relevant definitions referenced
by `v2/ThreadStartParams.json` and `v2/TurnStartParams.json`. Implementations MAY validate these
fields locally if they want stricter startup checks.

- `command` (string shell command)
  - Default: `codex app-server`
  - The runtime launches this command via `bash -lc` in the workspace directory.
  - The launched process MUST speak a compatible app-server protocol over stdio.
- `approval_policy` (Codex `AskForApproval` value)
  - Default: implementation-defined.
- `thread_sandbox` (Codex `SandboxMode` value)
  - Default: implementation-defined.
- `turn_sandbox_policy` (Codex `SandboxPolicy` value)
  - Default: implementation-defined.
- `turn_timeout_ms` (integer)
  - Default: `3600000` (1 hour)
- `read_timeout_ms` (integer)
  - Default: `5000`
- `stall_timeout_ms` (integer)
  - Default: `300000` (5 minutes)
  - If `<= 0`, stall detection is disabled.

#### 5.3.7 `verdict_state_map` (object, Recommended Extension)

When the worker writes a structured verdict (Section 10.4.1), the workflow can declare an explicit
verdict-to-state routing table instead of relying on the orchestrator inferring close-vs-handoff
from heuristics.

Schema:

- Map keys are verdict tokens drawn from a closed alphabet declared by the workflow (for example
  `DONE`, `APPROVE`, `UNABLE`, `AWAITING_ANSWERS`, `CHANGES_REQUESTED`, `SKIP`).
- Map values are tracker state names that must be members of `tracker.active_states`,
  `tracker.paused_states`, or `tracker.terminal_states`.
- Map values must NOT name a `tracker.backlog_states` entry — backlog is operator-only.
- Closing routes (values in `terminal_states`) should target `tracker.done_state` if it is set.

Validation:

- A verdict written by a worker that does not appear in the workflow's `verdict_state_map` should
  halt rather than fall back to a default. See Section 8.10 for the halt mechanism.
- Missing verdict files (`.symphony/verdict.txt` absent at end of run) should also halt — the
  orchestrator must not guess intent from exit code or commit count alone.

Rationale: a closed verdict alphabet plus an explicit routing table makes worker intent reviewable
in front matter and prevents silent state-machine drift when a new verdict appears in worker prompt
edits.

#### 5.3.8 `rules` (object, Recommended Extension)

A workflow may declare per-workflow worker constraints separately from the prompt body. The runtime
should surface these to the worker prompt and (where mechanically enforceable) check them at end of
run.

Schema:

- `must` (list of strings)
  - Imperative requirements (for example "Commit every file you create before exiting").
- `must_not` (list of strings)
  - Negative requirements (for example "Do not edit files outside `.symphony/`").
- `limits` (object, optional)
  - `max_diff_loc` (integer) — reject runs whose total diff exceeds this LOC count.
  - Other implementation-defined per-run hard caps.

Rules act as a compile-time-style worker contract: the prompt body is the design intent; the rules
block is the invariant the runtime should police.

### 5.4 Prompt Template Contract

The Markdown body of `WORKFLOW.md` is the per-issue prompt template.

Rendering requirements:

- Use a strict template engine (Liquid-compatible semantics are sufficient).
- Unknown variables MUST fail rendering.
- Unknown filters MUST fail rendering.

Template input variables:

- `issue` (object)
  - Includes all normalized issue fields, including labels and blockers.
- `attempt` (integer or null)
  - `null`/absent on first attempt.
  - Integer on retry or continuation run.

Fallback prompt behavior:

- If the workflow prompt body is empty, the runtime MAY use a minimal default prompt
  (`You are working on an issue from Linear.`).
- Workflow file read/parse failures are configuration/validation errors and SHOULD NOT silently fall
  back to a prompt.

### 5.5 Workflow Validation and Error Surface

Error classes:

- `missing_workflow_file`
- `workflow_parse_error`
- `workflow_front_matter_not_a_map`
- `template_parse_error` (during prompt rendering)
- `template_render_error` (unknown variable/filter, invalid interpolation)

Dispatch gating behavior:

- Workflow file read/YAML errors block new dispatches until fixed.
- Template errors fail only the affected run attempt.

### 5.6 Generated Workflow Alphabets (Recommended Extension)

Implementations using statically typed reducers may add a build step that reads each workflow's
front matter (`verdict_state_map`, `active_states`, `paused_states`, `terminal_states`) and emits a
type module per workflow exporting closed-union types for that workflow's verdicts and states.

Recommended properties:

- The codegen step runs as part of the normal build (for example `codegen:workflows` script run
  before `tsc`).
- Generated files live under a known directory (for example `src/reducer/generated/`) and are
  checked in so that diffs are reviewable.
- The reducer's switch statements over verdicts and states are exhaustiveness-checked at compile
  time against these unions.
- A new verdict appearing in a workflow's `verdict_state_map` shifts a build failure, not a runtime
  fallback. Operators see drift at PR time, not in production logs.

This is not required by the core spec — interpreted-language implementations may achieve similar
guarantees through runtime validation against the same front matter — but it is the recommended
pattern for typed implementations.

## 6. Configuration Specification

### 6.1 Configuration Resolution Pipeline

Configuration is resolved in this order:

1. Select the workflow file path (explicit runtime setting, otherwise cwd default).
2. Parse YAML front matter into a raw config map.
3. Apply built-in defaults for missing OPTIONAL fields.
4. Resolve `$VAR_NAME` indirection only for config values that explicitly contain `$VAR_NAME`.
5. Coerce and validate typed values.

Environment variables do not globally override YAML values. They are used only when a config value
explicitly references them.

Value coercion semantics:

- Path/command fields support:
  - `~` home expansion
  - `$VAR` expansion for env-backed path values
  - Apply expansion only to values intended to be local filesystem paths; do not rewrite URIs or
    arbitrary shell command strings.
- Relative `workspace.root` values resolve relative to the directory containing the selected
  `WORKFLOW.md`.

### 6.2 Dynamic Reload Semantics

Dynamic reload is REQUIRED:

- The software MUST detect `WORKFLOW.md` changes.
- On change, it MUST re-read and re-apply workflow config and prompt template without restart.
- The software MUST attempt to adjust live behavior to the new config (for example polling
  cadence, concurrency limits, active/terminal states, codex settings, workspace paths/hooks, and
  prompt content for future runs).
- Reloaded config applies to future dispatch, retry scheduling, reconciliation decisions, hook
  execution, and agent launches.
- Implementations are not REQUIRED to restart in-flight agent sessions automatically when config
  changes.
- Extensions that manage their own listeners/resources (for example an HTTP server port change) MAY
  require restart unless the implementation explicitly supports live rebind.
- Implementations SHOULD also re-validate/reload defensively during runtime operations (for example
  before dispatch) in case filesystem watch events are missed.
- Invalid reloads MUST NOT crash the service; keep operating with the last known good effective
  configuration and emit an operator-visible error.

### 6.3 Dispatch Preflight Validation

This validation is a scheduler preflight run before attempting to dispatch new work. It validates
the workflow/config needed to poll and launch workers, not a full audit of all possible workflow
behavior.

Startup validation:

- Validate configuration before starting the scheduling loop.
- If startup validation fails, fail startup and emit an operator-visible error.

Per-tick dispatch validation:

- Re-validate before each dispatch cycle.
- If validation fails, skip dispatch for that tick, keep reconciliation active, and emit an
  operator-visible error.

Validation checks:

- Workflow file can be loaded and parsed.
- `tracker.kind` is present and supported.
- `tracker.api_key` is present after `$` resolution.
- `tracker.project_slug` is present when REQUIRED by the selected tracker kind.
- `codex.command` is present and non-empty.

### 6.4 Core Config Fields Summary (Cheat Sheet)

This section is intentionally redundant so a coding agent can implement the config layer quickly.
Extension fields are documented in the extension section that defines them. Core conformance does
not require recognizing or validating extension fields unless that extension is implemented.

- `tracker.kind`: string, REQUIRED, currently `linear`
- `tracker.endpoint`: string, default `https://api.linear.app/graphql` when `tracker.kind=linear`
- `tracker.api_key`: string or `$VAR`, canonical env `LINEAR_API_KEY` when `tracker.kind=linear`
- `tracker.project_slug`: string, REQUIRED when `tracker.kind=linear`
- `tracker.active_states`: list of strings, default `["Todo", "In Progress"]`
- `tracker.terminal_states`: list of strings, default `["Closed", "Cancelled", "Canceled", "Duplicate", "Done"]`
- `polling.interval_ms`: integer, default `30000`
- `workspace.root`: path resolved to absolute, default `<system-temp>/symphony_workspaces`
- `hooks.after_create`: shell script or null
- `hooks.before_run`: shell script or null
- `hooks.after_run`: shell script or null
- `hooks.before_remove`: shell script or null
- `hooks.timeout_ms`: integer, default `60000`
- `agent.max_concurrent_agents`: integer, default `10`
- `agent.max_turns`: integer, default `20`
- `agent.max_retry_backoff_ms`: integer, default `300000` (5m)
- `agent.max_concurrent_agents_by_state`: map of positive integers, default `{}`
- `codex.command`: shell command string, default `codex app-server`
- `codex.approval_policy`: Codex `AskForApproval` value, default implementation-defined
- `codex.thread_sandbox`: Codex `SandboxMode` value, default implementation-defined
- `codex.turn_sandbox_policy`: Codex `SandboxPolicy` value, default implementation-defined
- `codex.turn_timeout_ms`: integer, default `3600000`
- `codex.read_timeout_ms`: integer, default `5000`
- `codex.stall_timeout_ms`: integer, default `300000`

## 7. Orchestration State Machine

The orchestrator is the only component that mutates scheduling state. All worker outcomes are
reported back to it and converted into explicit state transitions.

### 7.1 Issue Orchestration States

This is not the same as tracker states (`Todo`, `In Progress`, etc.). This is the service's internal
claim state.

1. `Unclaimed`
   - Issue is not running and has no retry scheduled.

2. `Claimed`
   - Orchestrator has reserved the issue to prevent duplicate dispatch.
   - In practice, claimed issues are either `Running` or `RetryQueued`.

3. `Running`
   - Worker task exists and the issue is tracked in `running` map.

4. `RetryQueued`
   - Worker is not running, but a retry timer exists in `retry_attempts`.

5. `Released`
   - Claim removed because issue is terminal, non-active, missing, or retry path completed without
     re-dispatch.

Important nuance:

- A successful worker exit does not mean the issue is done forever.
- The worker MAY continue through multiple back-to-back coding-agent turns before it exits.
- After each normal turn completion, the worker re-checks the tracker issue state.
- If the issue is still in an active state, the worker SHOULD start another turn on the same live
  coding-agent thread in the same workspace, up to `agent.max_turns`.
- The first turn SHOULD use the full rendered task prompt.
- Continuation turns SHOULD send only continuation guidance to the existing thread, not resend the
  original task prompt that is already present in thread history.
- Once the worker exits normally, the orchestrator still schedules a short continuation retry
  (about 1 second) so it can re-check whether the issue remains active and needs another worker
  session.

### 7.2 Run Attempt Lifecycle

A run attempt transitions through these phases:

1. `PreparingWorkspace`
2. `BuildingPrompt`
3. `LaunchingAgentProcess`
4. `InitializingSession`
5. `StreamingTurn`
6. `Finishing`
7. `Succeeded`
8. `Failed`
9. `TimedOut`
10. `Stalled`
11. `CanceledByReconciliation`

Distinct terminal reasons are important because retry logic and logs differ.

### 7.3 Transition Triggers

- `Poll Tick`
  - Reconcile active runs.
  - Validate config.
  - Fetch candidate issues.
  - Dispatch until slots are exhausted.

- `Worker Exit (normal)`
  - Remove running entry.
  - Update aggregate runtime totals.
  - Schedule continuation retry (attempt `1`) after the worker exhausts or finishes its in-process
    turn loop.

- `Worker Exit (abnormal)`
  - Remove running entry.
  - Update aggregate runtime totals.
  - Schedule exponential-backoff retry.

- `Codex Update Event`
  - Update live session fields, token counters, and rate limits.

- `Retry Timer Fired`
  - Re-fetch active candidates and attempt re-dispatch, or release claim if no longer eligible.

- `Reconciliation State Refresh`
  - Stop runs whose issue states are terminal or no longer active.

- `Stall Timeout`
  - Kill worker and schedule retry.

### 7.4 Idempotency and Recovery Rules

- The orchestrator serializes state mutations through one authority to avoid duplicate dispatch.
- `claimed` and `running` checks are REQUIRED before launching any worker.
- Reconciliation runs before dispatch on every tick.
- Restart recovery is tracker-driven and filesystem-driven (without a durable orchestrator DB).
- Startup terminal cleanup removes stale workspaces for issues already in terminal states.

### 7.5 Pure Reducer Architecture (Recommended Extension)

Implementations may split the orchestrator into a pure reducer and an effectful interpreter. The
reducer is a side-effect-free function `decide(state, event, spec) -> { state', effects[] }`; the
interpreter executes the effect array (tracker writes, subprocess launches, file writes, timers).

Properties of the pure reducer:

- No I/O. No clock. No randomness. No network. No filesystem. No environment access. No mutation
  of inputs.
- All inputs are arguments; all outputs are the returned tuple.
- Determinism: same `(state, event, spec)` always produces the same `(state', effects)`.

Why this is a recommended pattern:

- **Replayability**: a recorded event log (Section 13.9) plus the initial state reproduces the
  exact effect sequence in production. Forensic reconstruction does not require live infrastructure.
- **Property tests**: the reducer can be exercised at thousands of events per second under
  randomized inputs to catch state-machine drift before it ships.
- **Type-checked exhaustiveness**: combined with generated workflow alphabets (Section 5.6) the
  reducer's switch statements over verdicts and states are exhaustive at compile time.

Mechanical enforcement (recommended):

- Isolate the reducer module so it compiles without dependencies on tracker clients, subprocess
  helpers, or any runtime singleton.
- Add an import-boundary lint that rejects reducer imports of effectful modules.
- Add an identifier-boundary lint that rejects banned tokens inside the reducer file (`Date.now`,
  `Math.random`, `process.env`, `fetch`, `fs.`, `Octokit`, etc.).
- Run reducer property tests in CI as a gate.

Boundary contract: any module the reducer imports is also pure. The interpreter is the only place
side effects happen.

This pattern is the Tea / Elm Architecture applied to long-running orchestration. It is not
required by the core spec — implementations may use a more imperative orchestrator — but it makes
correctness arguments dramatically easier.

## 8. Polling, Scheduling, and Reconciliation

### 8.1 Poll Loop

At startup, the service validates config, performs startup cleanup, schedules an immediate tick, and
then repeats every `polling.interval_ms`.

The effective poll interval SHOULD be updated when workflow config changes are re-applied.

Tick sequence:

1. Reconcile running issues.
2. Run dispatch preflight validation.
3. Fetch candidate issues from tracker using active states.
4. Sort issues by dispatch priority.
5. Dispatch eligible issues while slots remain.
6. Notify observability/status consumers of state changes.

If per-tick validation fails, dispatch is skipped for that tick, but reconciliation still happens
first.

### 8.2 Candidate Selection Rules

An issue is dispatch-eligible only if all are true:

- It has `id`, `identifier`, `title`, and `state`.
- Its state is in `active_states` and not in `terminal_states`.
- It is not already in `running`.
- It is not already in `claimed`.
- Global concurrency slots are available.
- Per-state concurrency slots are available.
- Blocker rule for `Todo` state passes:
  - If the issue state is `Todo`, do not dispatch when any blocker is non-terminal.

Sorting order (stable intent):

1. `priority` ascending (1..4 are preferred; null/unknown sorts last)
2. `created_at` oldest first
3. `identifier` lexicographic tie-breaker

### 8.3 Concurrency Control

Global limit:

- `available_slots = max(max_concurrent_agents - running_count, 0)`

Per-state limit:

- `max_concurrent_agents_by_state[state]` if present (state key normalized)
- otherwise fallback to global limit

The runtime counts issues by their current tracked state in the `running` map.

### 8.4 Retry and Backoff

Retry entry creation:

- Cancel any existing retry timer for the same issue.
- Store `attempt`, `identifier`, `error`, `due_at_ms`, and new timer handle.

Backoff formula:

- Normal continuation retries after a clean worker exit use a short fixed delay of `1000` ms.
- Failure-driven retries use `delay = min(10000 * 2^(attempt - 1), agent.max_retry_backoff_ms)`.
- Power is capped by the configured max retry backoff (default `300000` / 5m).

Retry handling behavior:

1. Fetch active candidate issues (not all issues).
2. Find the specific issue by `issue_id`.
3. If not found, release claim.
4. If found and still candidate-eligible:
   - Dispatch if slots are available.
   - Otherwise requeue with error `no available orchestrator slots`.
5. If found but no longer active, release claim.

Note:

- Terminal-state workspace cleanup is handled by startup cleanup and active-run reconciliation
  (including terminal transitions for currently running issues).
- Retry handling mainly operates on active candidates and releases claims when the issue is absent,
  rather than performing terminal cleanup itself.

### 8.5 Active Run Reconciliation

Reconciliation runs every tick and has two parts.

Part A: Stall detection

- For each running issue, compute `elapsed_ms` since:
  - `last_codex_timestamp` if any event has been seen, else
  - `started_at`
- If `elapsed_ms > codex.stall_timeout_ms`, terminate the worker and queue a retry.
- If `stall_timeout_ms <= 0`, skip stall detection entirely.

Part B: Tracker state refresh

- Fetch current issue states for all running issue IDs.
- For each running issue:
  - If tracker state is terminal: terminate worker and clean workspace.
  - If tracker state is still active: update the in-memory issue snapshot.
  - If tracker state is neither active nor terminal: terminate worker without workspace cleanup.
- If state refresh fails, keep workers running and try again on the next tick.

### 8.6 Startup Terminal Workspace Cleanup

When the service starts:

1. Query tracker for issues in terminal states.
2. For each returned issue identifier, remove the corresponding workspace directory.
3. If the terminal-issues fetch fails, log a warning and continue startup.

This prevents stale terminal workspaces from accumulating after restarts.

### 8.7 Rate-Limit Gating (Recommended Extension)

When the coding agent emits rate-limit events with explicit timing hints, the orchestrator should
suppress new dispatches until the deadline passes. Running agents handle their own in-session
retries; only new dispatch is gated.

Gating semantics:

- Inspect rate-limit event payloads for explicit timing fields: `retry_after` (seconds),
  `reset_at` (ISO timestamp), or `reset` (epoch timestamp).
- When a timing hint is present and the associated status indicates exhaustion (for example
  `exceeded`, `exhausted`, `throttled`, `limited`), set a monotonic gate deadline.
- While `now < gate_deadline`, skip new dispatches in the tick loop.
- Informational rate-limit events without timing hints should be recorded for observability but
  should not trigger gating.
- If multiple rate-limit events arrive, the latest deadline wins (extend, never shorten).

This prevents thundering-herd behavior when multiple concurrent agents encounter API throttling
simultaneously.

### 8.8 Budget Guards (Recommended Extension)

The core spec defines retry delay caps (`agent.max_retry_backoff_ms`) and per-run turn caps
(`agent.max_turns`) but does not cap continuation count, failure retry count, or per-issue cost. A
worker that always exhausts its turn budget can loop indefinitely, and a worker that keeps failing
can retry forever with only the backoff delay bounded.

Implementations should consider adding per-issue budget caps:

- `agent.max_continuations` (integer, recommended default: `5`)
  - Caps the number of continuation cycles per issue. When a worker succeeds with
    `reachedTurnLimit=true`, increment a per-issue continuation counter. If it reaches the cap,
    release the claim and stop. The issue stays in its current tracker state for human review.
- `agent.max_failure_retries` (integer, recommended default: `3`)
  - Caps the number of failure retry attempts per issue. On each failure, increment a per-issue
    failure counter. If it reaches the cap, release the claim and stop.
- `agent.max_cost_per_issue_usd` (number, recommended default: `5.0`)
  - Cumulative dollar cap across all runs (continuations + retries + initial attempt) for one issue.
    Before scheduling a continuation or retry, check the budget; if exceeded, stop.

Per-issue counters should be tracked in a `issue_limits` map within runtime state. See Section
14.3.1 for persistence recommendations.

### 8.9 Working-State Dispatch Lock and Startup Reset (Recommended Extension)

When `tracker.working_state` is configured (Section 5.3.1):

Dispatch lock:

- On dispatch, the orchestrator should transition the issue to `working_state` in the tracker
  (best-effort, non-blocking).
- Candidate selection should exclude issues already in `working_state` unless they are claimed by the
  current orchestrator instance.
- This makes the in-memory claim visible in the tracker, acting as a distributed lock that survives
  restarts and is visible to other orchestrator instances sharing the same board.

Startup reset:

- Before the first tick, query the tracker for all issues in `working_state`.
- For each stale issue, transition it back to the first non-working active state (for example
  `Todo`) and post an audit comment noting the restart.
- Skip reset if `active_states` contains only the working state (no distinct ready state to reset
  to).
- Individual reset failures should be logged at warn level without aborting startup.

This prevents issues from being stuck in a "claimed but nobody is working on it" state after a crash
or restart.

### 8.10 Operator Halt / Andon Cord (Recommended Extension)

A halt mechanism lets operators (or the worker itself) freeze new dispatch when continuing would
cause more harm than waiting. Modeled on the Toyota Production System andon cord: any participant
can pull it; only a human can clear it.

Trigger sources:

- Worker writes a reserved verdict token (for example `HALT_PRODUCTION`) plus a reason file (for
  example `.symphony/halt-reason.txt`) at end of run.
- Orchestrator self-engages on classified failure modes (Section 8.10.1).
- Operator engages manually via a control surface (CLI, dashboard, or filesystem).

Lock representation:

- Recommended: a filesystem lock file at a known path (for example `<state_dir>/andon/<scope>.lock`
  for per-workflow halts, `<state_dir>/andon/global.lock` for fleet-wide halts).
- The lock file's contents are a structured record: trigger source, category (Section 8.10.1),
  reason text, ISO timestamp, originating issue identifier (when applicable).

Effect on dispatch:

- While the relevant lock is present, candidate selection skips dispatch for the matching scope.
- Active runs are not killed by halt engagement — running workers complete their turn and are not
  re-dispatched. Operators may opt to kill runs separately.
- The halt is observable in the snapshot API and the dashboard.

Clearing:

- No programmatic clear API by design. Operators remove the lock file (or run an audit-trail
  skill that removes it and posts a comment trail).
- This intentional friction prevents accidental clear-and-rerun loops.

#### 8.10.1 Halt Categories (Recommended Closed Union)

Halts should carry a category drawn from a closed set so dashboards can group and operators can
search. Recommended categories:

- `halt_production` — explicit worker- or operator-pulled halt
- `verdict_unknown` — worker wrote a verdict not in the workflow's `verdict_state_map`
- `verdict_missing` — worker exited without writing `.symphony/verdict.txt`
- `lease_invalid` — orchestrator detected a lease/epoch mismatch on a tracker write (Section
  11.6.1)
- `schema_drift` — config file references an alphabet member the build doesn't know about
- `rank_violation` — candidate selection produced an issue ineligible by `active_states`
- `quota_wall` — fleet-wide budget cap reached (Section 8.11)
- `agent_error` — repeated agent subprocess crashes within a rolling window
- `other` — escape hatch; operators should re-categorize after triage

Implementations may extend this union but should not repurpose existing categories.

### 8.11 Fleet-Wide Token Budget (Recommended Extension)

When a coding agent is billed by subscription tier rather than per-call (for example Claude
Code's session/weekly token quotas), a per-issue dollar cap (Section 8.8) is the wrong abstraction.
Implementations should consider a token-based fleet ledger that crosses workflow boundaries.

Schema:

- `fleet.tokens_cap_per_hour` (integer, optional)
- `fleet.tokens_cap_per_session` (integer, optional) — typical session window 5 hours
- `fleet.tokens_cap_per_week` (integer, optional)
- Storage: a single ledger file (for example `<state_dir>/fleet-budget-shared.json`) shared by all
  orchestrator processes for the fleet.
- Concurrency: cooperative POSIX file lock (`flock`) on the ledger file during read-modify-write.

Cap semantics:

- Each completed agent run records its consumed token count to the ledger.
- Pre-dispatch check sums tokens within each rolling window and refuses dispatch if any cap would
  be exceeded by a worst-case run.
- A breach engages a `quota_wall` halt (Section 8.10.1). Operators may raise the cap on a feature
  branch and merge; the orchestrator restart re-reads `fleet.json`.

Why token-based, not dollar-based:

- Subscription billing decouples per-token cost from monthly spend. Token caps reflect actual quota
  exhaustion; dollar caps invent a number that doesn't correspond to a cliff.
- Tier-aware presets (for example `MAX_5X` vs `MAX_20X` env vars) let the same code adapt to
  subscription changes without code edits.

### 8.12 Pre-Dispatch Guard Chain (Recommended Extension)

Every dispatch path should run a list of read-only guards before launching a worker. Each guard
returns a boolean: `true` blocks dispatch with a structured reason. The first guard returning
`true` halts the chain.

Guard list is a `ReadonlyArray<Guard>` registered at startup. Recommended starter guards:

- `AbandonedDesyncGuard` — refuses dispatch if the issue is in the persisted `abandoned` set
  (Section 14.3.1) but the tracker shows it ready. Intervention required.
- `LoopDetectorGuard` — counts dispatches per issue per rolling window and halts if a threshold is
  exceeded. Independent of the token ledger so loop detection works even when the ledger is broken.
- (Implementation-defined additional guards — for example git state assertions, lease validity
  pre-checks.)

Properties:

- Guards must be side-effect-free. They observe state; they do not mutate it.
- A blocking guard should record a halt event with the relevant category (`other` if no category
  fits).
- The guard list is data-driven so adding a new guard is one entry, not a code path edit.

## 9. Workspace Management and Safety

### 9.1 Workspace Layout

Workspace root:

- `workspace.root` (normalized absolute path)

Per-issue workspace path:

- `<workspace.root>/<sanitized_issue_identifier>`

Workspace persistence:

- Workspaces are reused across runs for the same issue.
- Successful runs do not auto-delete workspaces.

### 9.2 Workspace Creation and Reuse

Input: `issue.identifier`

Algorithm summary:

1. Sanitize identifier to `workspace_key`.
2. Compute workspace path under workspace root.
3. Ensure the workspace path exists as a directory.
4. Mark `created_now=true` only if the directory was created during this call; otherwise
   `created_now=false`.
5. If `created_now=true`, run `after_create` hook if configured.

Notes:

- This section does not assume any specific repository/VCS workflow.
- Workspace preparation beyond directory creation (for example dependency bootstrap, checkout/sync,
  code generation) is implementation-defined and is typically handled via hooks.

### 9.3 OPTIONAL Workspace Population (Implementation-Defined)

The spec does not require any built-in VCS or repository bootstrap behavior.

Implementations MAY populate or synchronize the workspace using implementation-defined logic and/or
hooks (for example `after_create` and/or `before_run`).

Failure handling:

- Workspace population/synchronization failures return an error for the current attempt.
- If failure happens while creating a brand-new workspace, implementations MAY remove the partially
  prepared directory.
- Reused workspaces SHOULD NOT be destructively reset on population failure unless that policy is
  explicitly chosen and documented.

### 9.4 Workspace Hooks

Supported hooks:

- `hooks.after_create`
- `hooks.before_run`
- `hooks.after_run`
- `hooks.before_remove`

Execution contract:

- Execute in a local shell context appropriate to the host OS, with the workspace directory as
  `cwd`.
- On POSIX systems, `sh -lc <script>` (or a stricter equivalent such as `bash -lc <script>`) is a
  conforming default.
- Hook timeout uses `hooks.timeout_ms`; default: `60000 ms`.
- Log hook start, failures, and timeouts.

Process group handling:

- Hook subprocesses should be spawned in a new process group (for example `detached: true` on
  POSIX) so that timeout kills terminate the entire process tree, not just the shell.
- On timeout, send `SIGTERM` to the negative PID (process group) rather than to the shell process
  alone. Fall back to direct kill if the process group is already gone.
- Without this, long-running child processes (for example `git clone`, dependency installs) survive
  the hook timeout and leak.

Hook environment variables:

- `before_run` and `after_run` hooks should receive a `SYMPHONY_ATTEMPT` environment variable
  indicating the current run attempt: `"initial"` on the first run for an issue, or a positive
  integer string (`"1"`, `"2"`, ...) on continuations and retries.
- This allows hooks to preserve in-progress state across continuation turns (for example skipping
  a fresh `git fetch` when scratch files from a prior turn are still valid) instead of
  unconditionally resetting the workspace.

Failure semantics:

- `after_create` failure or timeout is fatal to workspace creation.
- `before_run` failure or timeout is fatal to the current run attempt.
- `after_run` failure or timeout is logged and ignored.
- `before_remove` failure or timeout is logged and ignored.

### 9.5 Safety Invariants

This is the most important portability constraint.

Invariant 1: Run the coding agent only in the per-issue workspace path.

- Before launching the coding-agent subprocess, validate:
  - `cwd == workspace_path`

Invariant 2: Workspace path MUST stay inside workspace root.

- Normalize both paths to absolute.
- Require `workspace_path` to have `workspace_root` as a prefix directory.
- Reject any path outside the workspace root.

Invariant 3: Workspace key is sanitized.

- Only `[A-Za-z0-9._-]` allowed in workspace directory names.
- Replace all other characters with `_`.

### 9.6 Workspace Preservation on Abandonment (Recommended Extension)

When the orchestrator decides to stop work on an issue mid-flight (operator moved the card to a
backlog state, lease invalidated, shutdown drain), it should preserve any uncommitted or unpushed
worker work before tearing down the workspace, not discard it via `git reset` or `git clean`.

Recommended preservation flow:

1. Detect uncommitted changes (`git status --porcelain`) and unpushed commits (`git rev-list
   @{u}..HEAD`).
2. If anything is found, create a preservation branch `abandoned/<original-branch>-<timestamp>` and
   force-push it to the remote.
3. Optionally post a tracker comment linking the preservation branch.
4. Then proceed with workspace teardown.

Why:

- Worker work that the orchestrator decides not to merge may still contain insight (a partial
  implementation, an approach the operator wants to reuse). Discarding it because the orchestrator
  changed its mind is destructive.
- A `git reset --hard` is irreversible from outside the orchestrator. A preservation branch is
  reversible and observable.

Failures of preservation should be logged and should not block teardown — preservation is
best-effort, not a correctness gate.

### 9.7 Tick Watchdog and External Supervision (Recommended Extension)

A long-running orchestrator can hang on a stuck syscall, deadlock, or infinite loop without ever
crashing. Implementations should consider two layers of liveness defense:

In-process tick watchdog:

- Each tick has a deadline (recommended default: 90s).
- A separate timer fires if the deadline passes; on fire, the orchestrator emits a structured
  `TICK_TIMEOUT` event and exits with a distinguished exit code.
- Tracker calls (and other long syscalls) carry their own per-request timeout and emit
  `STALL_OBSERVED` reducer events on overrun. N stalls in a rolling window engage an `agent_error`
  halt.

External supervisor:

- The orchestrator runs under a process supervisor (launchd on macOS, systemd on Linux) with an
  automatic respawn policy and a throttle interval (for example minimum 30s between respawns).
- A separate reaper script may be used as a kill-switch: `pkill -TERM -- -$PGID` (negative PID =
  process group) to terminate the entire process tree, not just the parent. This handles cases
  where in-process timers themselves are stuck.

Properties:

- The orchestrator should write its PID and PGID to a known file at startup so the reaper script
  can find them deterministically.
- The supervisor's job is liveness; the orchestrator's job is correctness. Don't conflate them by
  putting "is the work valid?" logic in the supervisor or "should we restart?" logic in the
  orchestrator.

## 10. Agent Runner Protocol (Coding Agent Integration)

This section defines the language-neutral contract for integrating a coding agent. The primary
example uses Codex's app-server JSON-RPC protocol, and the Codex app-server protocol for the
targeted Codex version is the source of truth for protocol schemas, message payloads, transport
framing, and method names. The logical contract applies to any coding agent that can run as a
subprocess and produce structured output.

Implementations using CLI-based agents (for example Claude Code with `--output-format stream-json`)
that do not use a JSON-RPC app-server protocol MAY adapt the contracts below to their agent's
integration model. The essential requirements are: (1) launch the agent in the workspace directory,
(2) stream structured events back to the orchestrator, (3) detect turn completion/failure, and (4)
support session continuation across turns.

Protocol source of truth:

- Implementations MUST send messages that are valid for the targeted Codex app-server version.
- Implementations MUST consult the targeted Codex app-server documentation or generated schema
  instead of treating this specification as a protocol schema.
- If this specification appears to conflict with the targeted Codex app-server protocol, the Codex
  protocol controls protocol shape and transport behavior.
- Symphony-specific requirements in this section still control orchestration behavior, workspace
  selection, prompt construction, continuation handling, and observability extraction.

### 10.1 Launch Contract

Subprocess launch parameters:

- Command: `codex.command`
- Invocation: `bash -lc <codex.command>`
- Working directory: workspace path
- Transport/framing: the protocol transport required by the targeted Codex app-server version

Notes:

- The default command is `codex app-server`.
- Approval policy, sandbox policy, cwd, prompt input, and OPTIONAL tool declarations are supplied
  using fields supported by the targeted Codex app-server version (see the protocol messages in
  Section 10.2).
- CLI-based agents that accept prompt and options as command-line arguments (for example
  `claude --print --output-format stream-json -p <prompt>`) MAY be launched directly without
  `bash -lc` wrapping. The working directory constraint still applies.

RECOMMENDED additional process settings:

- Max line size: 10 MB (for safe buffering)

### 10.2 Session Startup Responsibilities

Reference: https://developers.openai.com/codex/app-server/

Startup MUST follow the targeted Codex app-server contract. Symphony additionally requires the
client to:

- Start the app-server subprocess in the per-issue workspace.
- Initialize the app-server session using the targeted Codex app-server protocol.
- Create or resume a coding-agent thread according to the targeted protocol.
- Supply the absolute per-issue workspace path as the thread/turn working directory wherever the
  targeted protocol accepts cwd.
- Start the first turn with the rendered issue prompt.
- Start later in-worker continuation turns on the same live thread with continuation guidance rather
  than resending the original issue prompt.
- Supply the implementation's documented approval and sandbox policy using fields supported by the
  targeted protocol.
- Include issue-identifying metadata, such as `<issue.identifier>: <issue.title>`, when the targeted
  protocol supports turn or session titles.
- Advertise implemented client-side tools using the targeted protocol.

Note: CLI-based agents that do not use a JSON-RPC app-server protocol MAY skip the handshake
entirely. The session starts implicitly when the subprocess begins producing output. Session
identifiers SHOULD be extracted from the agent's output stream (for example a `session_id` field in
the result event).

Session identifiers:

- Extract `thread_id` from the thread identity returned by the targeted Codex app-server protocol.
- Extract `turn_id` from each turn identity returned by the targeted Codex app-server protocol.
- Emit `session_id = "<thread_id>-<turn_id>"`
- Reuse the same `thread_id` for all continuation turns inside one worker run

### 10.3 Streaming Turn Processing

The client processes app-server updates according to the targeted Codex app-server protocol until
the active turn terminates.

Completion conditions:

- Targeted-protocol turn completion signal -> success
- Targeted-protocol turn failure signal -> failure
- Targeted-protocol turn cancellation signal -> failure
- turn timeout (`turn_timeout_ms`) -> failure
- subprocess exit -> failure

Continuation processing:

- If the worker decides to continue after a successful turn, it SHOULD start another turn on the same
  live thread using the targeted protocol.
- The app-server subprocess SHOULD remain alive across those continuation turns and be stopped only
  when the worker run is ending.
- Alternative for CLI-based agents: spawn a new subprocess with a session-resume flag (for example
  `--resume <sessionId>`) instead of issuing additional turns on a live subprocess. The agent's
  result event should include a `reachedTurnLimit` flag (or equivalent) so the orchestrator knows
  whether to schedule a continuation.

Transport handling requirements:

- Follow the transport and framing rules of the targeted Codex app-server version.
- For stdio-based transports, keep protocol stream handling separate from diagnostic stderr
  handling unless the targeted protocol specifies otherwise.

### 10.4 Emitted Runtime Events (Upstream to Orchestrator)

The app-server client emits structured events to the orchestrator callback. Each event SHOULD
include:

- `event` (enum/string)
- `timestamp` (UTC timestamp)
- `codex_app_server_pid` (if available)
- OPTIONAL `usage` map (token counts)
- payload fields as needed

Important emitted events include, for example:

- `session_started`
- `startup_failed`
- `turn_completed`
- `turn_failed`
- `turn_cancelled`
- `turn_ended_with_error`
- `turn_input_required`
- `approval_auto_approved`
- `unsupported_tool_call`
- `notification`
- `other_message`
- `malformed`

### 10.5 Approval, Tool Calls, and User Input Policy

Approval, sandbox, and user-input behavior is implementation-defined.

Policy requirements:

- Each implementation MUST document its chosen approval, sandbox, and operator-confirmation
  posture.
- Approval requests and user-input-required events MUST NOT leave a run stalled indefinitely. An
  implementation MAY either satisfy them, surface them to an operator, auto-resolve them, or
  fail the run according to its documented policy.

Example high-trust behavior:

- Auto-approve command execution approvals for the session.
- Auto-approve file-change approvals for the session.
- Treat user-input-required turns as hard failure.

Unsupported dynamic tool calls:

- Supported dynamic tool calls that are explicitly implemented and advertised by the runtime SHOULD
  be handled according to their extension contract.
- If the agent requests a dynamic tool call that is not supported, return a tool failure response
  using the targeted protocol and continue the session.
- This prevents the session from stalling on unsupported tool execution paths.

Optional client-side tool extension:

- An implementation MAY expose a limited set of client-side tools to the app-server session.
- Current standardized optional tool: `linear_graphql`. Implementations using a different tracker
  SHOULD define an equivalent tool appropriate to their backend (for example `github_graphql` for
  GitHub-based implementations).
- If implemented, supported tools SHOULD be advertised to the app-server session during startup
  using the protocol mechanism supported by the targeted Codex app-server version.
- Unsupported tool names SHOULD still return a failure result using the targeted protocol and
  continue the session.

`linear_graphql` extension contract:

- Purpose: execute a raw GraphQL query or mutation against Linear using Symphony's configured
  tracker auth for the current session.
- Availability: only meaningful when `tracker.kind == "linear"` and valid Linear auth is configured.
- Preferred input shape:

  ```json
  {
    "query": "single GraphQL query or mutation document",
    "variables": {
      "optional": "graphql variables object"
    }
  }
  ```

- `query` MUST be a non-empty string.
- `query` MUST contain exactly one GraphQL operation.
- `variables` is OPTIONAL and, when present, MUST be a JSON object.
- Implementations MAY additionally accept a raw GraphQL query string as shorthand input.
- Execute one GraphQL operation per tool call.
- If the provided document contains multiple operations, reject the tool call as invalid input.
- `operationName` selection is intentionally out of scope for this extension.
- Reuse the configured Linear endpoint and auth from the active Symphony workflow/runtime config; do
  not require the coding agent to read raw tokens from disk.
- Tool result semantics:
  - transport success + no top-level GraphQL `errors` -> `success=true`
  - top-level GraphQL `errors` present -> `success=false`, but preserve the GraphQL response body
    for debugging
  - invalid input, missing auth, or transport failure -> `success=false` with an error payload
- Return the GraphQL response or error payload as structured tool output that the model can inspect
  in-session.

User-input-required policy:

- Implementations MUST document how targeted-protocol user-input-required signals are handled.
- A run MUST NOT stall indefinitely waiting for user input.
- A conforming implementation MAY fail the run, surface the request to an operator, satisfy it
  through an approved operator channel, or auto-resolve it according to its documented policy.
- The example high-trust behavior above fails user-input-required turns immediately.

### 10.6 Timeouts and Error Mapping

Timeouts:

- `codex.read_timeout_ms`: request/response timeout during startup and sync requests
- `codex.turn_timeout_ms`: total turn stream timeout
- `codex.stall_timeout_ms`: enforced by orchestrator based on event inactivity

Error mapping (RECOMMENDED normalized categories):

- `codex_not_found`
- `invalid_workspace_cwd`
- `response_timeout`
- `turn_timeout`
- `port_exit`
- `response_error`
- `turn_failed`
- `turn_cancelled`
- `turn_input_required`

### 10.7 Agent Runner Contract

The `Agent Runner` wraps workspace + prompt + app-server client.

Behavior:

1. Create/reuse workspace for issue.
2. Build prompt from workflow template.
3. Start app-server session.
4. Forward app-server events to orchestrator.
5. On any error, fail the worker attempt (the orchestrator will retry).

Note:

- Workspaces are intentionally preserved after successful runs.

## 11. Issue Tracker Integration Contract

### 11.1 REQUIRED Operations

An implementation MUST support these tracker adapter operations:

1. `fetch_candidate_issues()`
   - Return issues in configured active states for a configured project.

2. `fetch_issues_by_states(state_names)`
   - Used for startup terminal cleanup.

3. `fetch_issue_states_by_ids(issue_ids)`
   - Used for active-run reconciliation.

Implementations may add optional write operations for audit trail and distributed-lock purposes. See
Section 11.6 for recommended tracker write extensions.

### 11.2 Query Semantics (Linear)

Linear-specific requirements for `tracker.kind == "linear"`:

- `tracker.kind == "linear"`
- GraphQL endpoint (default `https://api.linear.app/graphql`)
- Auth token sent in `Authorization` header
- `tracker.project_slug` maps to Linear project `slugId`
- Candidate issue query filters project using `project: { slugId: { eq: $projectSlug } }`
- Issue-state refresh query uses GraphQL issue IDs with variable type `[ID!]`
- Pagination REQUIRED for candidate issues
- Page size default: `50`
- Network timeout: `30000 ms`

Important:

- Linear GraphQL schema details can drift. Keep query construction isolated and test the exact query
  fields/types REQUIRED by this specification.

A non-Linear implementation MAY change transport details, but the normalized outputs MUST match the
domain model in Section 4.

### 11.2b Query Semantics (GitHub Projects v2)

GitHub-specific requirements for `tracker.kind == "github"`:

- GraphQL endpoint: `https://api.github.com/graphql`
- Auth token sent in `Authorization: Bearer` header
- `tracker.project_slug` is in `owner/repo` format
- `tracker.project_number` identifies the GitHub Projects v2 board
- `tracker.status_field` names the single-select Status field on the board
- Candidate issue query fetches all project items and filters by the Status field value client-side
- Issue identification uses GitHub node IDs for GraphQL mutations and numeric issue/PR numbers for
  REST operations
- For typical board sizes (under 500 items), client-side filtering after a single paginated fetch is
  acceptable. Larger boards may require server-side filtering if GitHub's Projects API supports it.
- Network timeout: `30000 ms`

Important:

- GitHub Projects v2 uses a different data model than Linear: status is a custom single-select field
  rather than a built-in state property. The implementation must resolve the field ID and option IDs
  at startup (cacheable) before performing status reads or writes.
- GitHub issues and pull requests share the same Projects v2 board. The `Issue` model's `url` field
  and normalized `identifier` (for example `owner/repo#123`) should distinguish between them.

### 11.3 Normalization Rules

Candidate issue normalization SHOULD produce fields listed in Section 4.1.1.

Additional normalization details:

- `labels` -> lowercase strings
- `blocked_by` -> derived from inverse relations where relation type is `blocks`
- `priority` -> integer only (non-integers become null)
- `created_at` and `updated_at` -> parse ISO-8601 timestamps

### 11.4 Error Handling Contract

RECOMMENDED error categories (generic):

- `unsupported_tracker_kind`
- `missing_tracker_api_key`
- `missing_tracker_project_slug`
- `tracker_api_request` (transport failures)
- `tracker_api_status` (non-200 HTTP)
- `tracker_graphql_errors`
- `tracker_unknown_payload`
- `tracker_missing_pagination_cursor` (pagination integrity error)

Implementations may use tracker-kind-specific prefixes (for example `linear_api_request`,
`github_api_request`) for more precise error routing.

Orchestrator behavior on tracker errors:

- Candidate fetch failure: log and skip dispatch for this tick.
- Running-state refresh failure: log and keep active workers running.
- Startup terminal cleanup failure: log warning and continue startup.

### 11.5 Tracker Writes (Important Boundary)

Symphony does not require first-class tracker write APIs in the orchestrator.

- Ticket mutations (state transitions, comments, PR metadata) are typically handled by the coding
  agent using tools defined by the workflow prompt.
- The service remains a scheduler/runner and tracker reader.
- Workflow-specific success often means "reached the next handoff state" (for example
  `Human Review`) rather than tracker terminal state `Done`.
- If the `linear_graphql` client-side tool extension is implemented, it is still part of the agent
  toolchain rather than orchestrator business logic.

### 11.6 Orchestrator-Driven Tracker Writes (Recommended Extension)

While the baseline treats the orchestrator as read-only (Section 11.5), implementations may add
write operations to the tracker adapter for audit trail and distributed-lock purposes.

Recommended optional operations:

1. `update_issue_state(issue_id, new_state)`
   - Used on dispatch to transition the issue to a working state (Section 8.9).
   - Used on completion to transition to a terminal or handoff state.

2. `post_comment(issue_id, body)`
   - Used to leave an audit trail on the issue when the orchestrator makes state decisions.
   - Recommended comment convention: `Symphony: -> **NewState** (reason)`.

3. `close_issue(issue_id)`
   - Used on successful completion when the workflow defines done as issue closure.

All tracker writes should be best-effort and non-blocking. Write failures should be logged but must
not block orchestrator operation or worker dispatch. The orchestrator remains a scheduler/runner
first; tracker writes are an observability and coordination enhancement, not a correctness
requirement.

#### 11.6.1 Optimistic Concurrency for Tracker Writes (Recommended Sub-Extension)

When the orchestrator writes to the tracker for an issue it claimed earlier, a second orchestrator
process (or a stale worker that was killed and re-dispatched) may try to write conflicting state.
Implementations should consider an epoch / lease token to detect this.

Recommended scheme:

- On dispatch, the orchestrator mints an `epoch` token (monotonic counter or UUID) and persists it
  on the tracker (for example as a `symphony:epoch:N` label or a custom field).
- All effects that mutate tracker state (state transition, close, sub-issue creation) carry an
  `expectedEpoch`.
- The tracker write reads the live epoch first and refuses the write if it differs from
  `expectedEpoch` (CAS pattern). Refused writes raise an `EPOCH_CONFLICT` error and engage a
  `lease_invalid` halt (Section 8.10.1).
- The epoch state on the tracker is production state — operators must not "clean up" stale epoch
  labels because they encode the lock witness.

This is optional but strongly recommended for any deployment running more than one orchestrator
process against the same board, or where worker subprocesses can outlive their orchestrator.

## 12. Prompt Construction and Context Assembly

### 12.1 Inputs

Inputs to prompt rendering:

- `workflow.prompt_template`
- normalized `issue` object
- OPTIONAL `attempt` integer (retry/continuation metadata)

### 12.2 Rendering Rules

- Render with strict variable checking.
- Render with strict filter checking.
- Convert issue object keys to strings for template compatibility.
- Preserve nested arrays/maps (labels, blockers) so templates can iterate.

### 12.3 Retry/Continuation Semantics

`attempt` SHOULD be passed to the template because the workflow prompt can provide different
instructions for:

- first run (`attempt` null or absent)
- continuation run after a successful prior session
- retry after error/timeout/stall

### 12.4 Failure Semantics

If prompt rendering fails:

- Fail the run attempt immediately.
- Let the orchestrator treat it like any other worker failure and decide retry behavior.

## 13. Logging, Status, and Observability

### 13.1 Logging Conventions

REQUIRED context fields for issue-related logs:

- `issue_id`
- `issue_identifier`

REQUIRED context for coding-agent session lifecycle logs:

- `session_id`

Message formatting requirements:

- Use stable `key=value` phrasing.
- Include action outcome (`completed`, `failed`, `retrying`, etc.).
- Include concise failure reason when present.
- Avoid logging large raw payloads unless necessary.

### 13.2 Logging Outputs and Sinks

The spec does not prescribe where logs are written (stderr, file, remote sink, etc.).

Requirements:

- Operators MUST be able to see startup/validation/dispatch failures without attaching a debugger.
- Implementations MAY write to one or more sinks.
- If a configured log sink fails, the service SHOULD continue running when possible and emit an
  operator-visible warning through any remaining sink.

### 13.3 Runtime Snapshot / Monitoring Interface (OPTIONAL but RECOMMENDED)

If the implementation exposes a synchronous runtime snapshot (for dashboards or monitoring), it
SHOULD return:

- `running` (list of running session rows)
- each running row SHOULD include `turn_count`
- `retrying` (list of retry queue rows)
- `codex_totals`
  - `input_tokens`
  - `output_tokens`
  - `total_tokens`
  - `seconds_running` (aggregate runtime seconds as of snapshot time, including active sessions)
- `rate_limits` (latest coding-agent rate limit payload, if available)

RECOMMENDED snapshot error modes:

- `timeout`
- `unavailable`

### 13.4 OPTIONAL Human-Readable Status Surface

A human-readable status surface (terminal output, dashboard, etc.) is OPTIONAL and
implementation-defined.

If present, it SHOULD draw from orchestrator state/metrics only and MUST NOT be REQUIRED for
correctness.

### 13.5 Session Metrics and Token Accounting

Token accounting rules:

- Agent events can include token counts in multiple payload shapes.
- Prefer absolute thread totals when available, such as:
  - `thread/tokenUsage/updated` payloads
  - `total_token_usage` within token-count wrapper events
- Ignore delta-style payloads such as `last_token_usage` for dashboard/API totals.
- Extract input/output/total token counts leniently from common field names within the selected
  payload.
- For absolute totals, track deltas relative to last reported totals to avoid double-counting.
- Do not treat generic `usage` maps as cumulative totals unless the event type defines them that
  way.
- Accumulate aggregate totals in orchestrator state.

Runtime accounting:

- Runtime SHOULD be reported as a live aggregate at snapshot/render time.
- Implementations MAY maintain a cumulative counter for ended sessions and add active-session
  elapsed time derived from `running` entries (for example `started_at`) when producing a
  snapshot/status view.
- Add run duration seconds to the cumulative ended-session runtime when a session ends (normal exit
  or cancellation/termination).
- Continuous background ticking of runtime totals is not REQUIRED.

Rate-limit tracking:

- Track the latest rate-limit payload seen in any agent update.
- Any human-readable presentation of rate-limit data is implementation-defined.

### 13.6 Humanized Agent Event Summaries (OPTIONAL)

Humanized summaries of raw agent protocol events are OPTIONAL.

If implemented:

- Treat them as observability-only output.
- Do not make orchestrator logic depend on humanized strings.

### 13.7 OPTIONAL HTTP Server Extension

This section defines an OPTIONAL HTTP interface for observability and operational control.

If implemented:

- The HTTP server is an extension and is not REQUIRED for conformance.
- The implementation MAY serve server-rendered HTML or a client-side application for the dashboard.
- The dashboard/API MUST be observability/control surfaces only and MUST NOT become REQUIRED for
  orchestrator correctness.

Extension config:

- `server.port` (integer, OPTIONAL)
  - Enables the HTTP server extension.
  - `0` requests an ephemeral port for local development and tests.
  - CLI `--port` overrides `server.port` when both are present.

Enablement (extension):

- Start the HTTP server when a CLI `--port` argument is provided.
- Start the HTTP server when `server.port` is present in `WORKFLOW.md` front matter.
- The `server` top-level key is owned by this extension.
- Positive `server.port` values bind that port.
- Implementations SHOULD bind loopback by default (`127.0.0.1` or host equivalent) unless explicitly
  configured otherwise.
- Changes to HTTP listener settings (for example `server.port`) do not need to hot-rebind;
  restart-required behavior is conformant.

#### 13.7.1 Human-Readable Dashboard (`/`)

- Host a human-readable dashboard at `/`.
- The returned document SHOULD depict the current state of the system (for example active sessions,
  retry delays, token consumption, runtime totals, recent events, and health/error indicators).
- It is up to the implementation whether this is server-generated HTML or a client-side app that
  consumes the JSON API below.

#### 13.7.2 JSON REST API (`/api/v1/*`)

Provide a JSON REST API under `/api/v1/*` for current runtime state and operational debugging.

Minimum endpoints:

- `GET /api/v1/state`
  - Returns a summary view of the current system state (running sessions, retry queue/delays,
    aggregate token/runtime totals, latest rate limits, and any additional tracked summary fields).
  - Suggested response shape:

    ```json
    {
      "generated_at": "2026-02-24T20:15:30Z",
      "counts": {
        "running": 2,
        "retrying": 1
      },
      "running": [
        {
          "issue_id": "abc123",
          "issue_identifier": "MT-649",
          "state": "In Progress",
          "session_id": "thread-1-turn-1",
          "turn_count": 7,
          "last_event": "turn_completed",
          "last_message": "",
          "started_at": "2026-02-24T20:10:12Z",
          "last_event_at": "2026-02-24T20:14:59Z",
          "tokens": {
            "input_tokens": 1200,
            "output_tokens": 800,
            "total_tokens": 2000
          }
        }
      ],
      "retrying": [
        {
          "issue_id": "def456",
          "issue_identifier": "MT-650",
          "attempt": 3,
          "due_at": "2026-02-24T20:16:00Z",
          "error": "no available orchestrator slots"
        }
      ],
      "codex_totals": {
        "input_tokens": 5000,
        "output_tokens": 2400,
        "total_tokens": 7400,
        "seconds_running": 1834.2
      },
      "rate_limits": null
    }
    ```

- `GET /api/v1/<issue_identifier>`
  - Returns issue-specific runtime/debug details for the identified issue, including any information
    the implementation tracks that is useful for debugging.
  - Suggested response shape:

    ```json
    {
      "issue_identifier": "MT-649",
      "issue_id": "abc123",
      "status": "running",
      "workspace": {
        "path": "/tmp/symphony_workspaces/MT-649"
      },
      "attempts": {
        "restart_count": 1,
        "current_retry_attempt": 2
      },
      "running": {
        "session_id": "thread-1-turn-1",
        "turn_count": 7,
        "state": "In Progress",
        "started_at": "2026-02-24T20:10:12Z",
        "last_event": "notification",
        "last_message": "Working on tests",
        "last_event_at": "2026-02-24T20:14:59Z",
        "tokens": {
          "input_tokens": 1200,
          "output_tokens": 800,
          "total_tokens": 2000
        }
      },
      "retry": null,
      "logs": {
        "codex_session_logs": [
          {
            "label": "latest",
            "path": "/var/log/symphony/codex/MT-649/latest.log",
            "url": null
          }
        ]
      },
      "recent_events": [
        {
          "at": "2026-02-24T20:14:59Z",
          "event": "notification",
          "message": "Working on tests"
        }
      ],
      "last_error": null,
      "tracked": {}
    }
    ```

  - If the issue is unknown to the current in-memory state, return `404` with an error response (for
    example `{\"error\":{\"code\":\"issue_not_found\",\"message\":\"...\"}}`).

- `POST /api/v1/refresh`
  - Queues an immediate tracker poll + reconciliation cycle (best-effort trigger; implementations
    MAY coalesce repeated requests).
  - Suggested request body: empty body or `{}`.
  - Suggested response (`202 Accepted`) shape:

    ```json
    {
      "queued": true,
      "coalesced": false,
      "requested_at": "2026-02-24T20:15:30Z",
      "operations": ["poll", "reconcile"]
    }
    ```

API design notes:

- The JSON shapes above are the RECOMMENDED baseline for interoperability and debugging ergonomics.
- Implementations MAY add fields, but SHOULD avoid breaking existing fields within a version.
- Endpoints SHOULD be read-only except for operational triggers like `/refresh`.
- Unsupported methods on defined routes SHOULD return `405 Method Not Allowed`.
- API errors SHOULD use a JSON envelope such as `{"error":{"code":"...","message":"..."}}`.
- If the dashboard is a client-side app, it SHOULD consume this API rather than duplicating state
  logic.

#### 13.7.3 Server-Sent Events (SSE) Extension

Implementations may expose SSE endpoints for real-time dashboard updates alongside the JSON REST API:

- `GET /api/v1/stream/state`
  - Broadcasts a full state snapshot on orchestrator state mutations.
  - Should send an initial snapshot immediately on connect so clients render without waiting for the
    next mutation.
  - Burst mutations should be debounced (for example 200ms) to avoid flooding clients during busy
    ticks.

- `GET /api/v1/stream/<issue_identifier>`
  - Streams per-issue agent log events in real time.
  - Should send buffered recent events on connect, then live events filtered by issue ID.

SSE headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`.

Event format should use the standard SSE `data:` field with JSON payloads.

### 13.8 Per-Session Structured Logging (Recommended Extension)

Implementations may write one structured log file per issue per attempt to enable post-mortem
analysis without parsing shared logs.

Recommended format: NDJSON (newline-delimited JSON), one event per line.

File naming convention:

- Path: `<log_dir>/<sanitized_identifier>-<iso_timestamp>.ndjson`
- Identifier sanitization: replace non-`[A-Za-z0-9._-]` characters with `_`
- Timestamp format should be path-safe (no `:` or `.`)

Each agent event (tool use, tool result, rate limit, progress, result) should be appended as a
standalone JSON line with at minimum:

- `timestamp` (ISO-8601)
- `type` (event type string)
- `message` (human-readable summary)

The orchestrator should write a summary record at session end with outcome reason, total cost, and
budget guard state.

All writes should be best-effort — log file errors should never break the orchestrator.

### 13.9 Reducer Event Log (Recommended Extension)

When implementations adopt the pure reducer architecture (Section 7.5), they should also write an
append-only event log of every reducer event and the effects it produced. This is the audit
substrate for replay and forensic reconstruction.

Format: NDJSON, one record per line, written by the interpreter (not the reducer).

Each record should include:

- `timestamp` (ISO-8601)
- `event` (the input event type and payload)
- `prev_state_summary` (a stable hash or compact projection — full state may be too large)
- `effects` (the array of effects emitted by the reducer for this event)
- `effect_outcomes` (the result of each effect — succeeded, failed with error class)

Properties:

- The log is append-only. Rotation is acceptable (size or time-based) but old segments must be
  retained for the replay window.
- Replay tooling consumes this log to reconstruct decisions in a sandbox without live tracker or
  agent calls.
- Per-workflow files prevent cross-workflow noise; per-issue files within a workflow are also
  acceptable.

This log is distinct from per-session NDJSON (Section 13.8): per-session captures the agent's
output stream; the reducer event log captures the orchestrator's decision stream.

### 13.10 Operator Intervention Log (Recommended Extension)

Operator overrides issued through CLI skills, dashboard buttons, or direct tracker mutations should
be recorded separately from worker-driven transitions so the audit envelope distinguishes
human-driven from agent-driven state changes.

Recommended format: NDJSON at `<state_dir>/interventions.ndjson`, one record per intervention.

Recommended fields:

- `timestamp` (ISO-8601)
- `actor` (identifier for the operator — username, email, or "system" for automated overrides)
- `kind` (closed union: `close`, `defer`, `revive`, `priority_change`, `halt_clear`,
  `epoch_reset`, `other`)
- `subject` (the issue identifier, PR number, or scope key affected)
- `reason` (free-text justification supplied by the operator)
- `metadata` (kind-specific payload — for example old/new priority for `priority_change`)

Properties:

- Append-only; never edited or truncated.
- Reviewable by humans (it is the diff between what the orchestrator wanted and what humans did).
- Source data for weekly autonomy/intervention reports.

Skills that mutate state outside the orchestrator's normal flow (close, defer, halt clear) should
write to this log as part of their own atomic flow, not depend on the orchestrator picking up the
mutation later.

## 14. Failure Model and Recovery Strategy

### 14.1 Failure Classes

1. `Workflow/Config Failures`
   - Missing `WORKFLOW.md`
   - Invalid YAML front matter
   - Unsupported tracker kind or missing tracker credentials/project slug
   - Missing coding-agent executable

2. `Workspace Failures`
   - Workspace directory creation failure
   - Workspace population/synchronization failure (implementation-defined; can come from hooks)
   - Invalid workspace path configuration
   - Hook timeout/failure

3. `Agent Session Failures`
   - Startup handshake failure
   - Turn failed/cancelled
   - Turn timeout
   - User input requested and handled as failure by the implementation's documented policy
   - Subprocess exit
   - Stalled session (no activity)

4. `Tracker Failures`
   - API transport errors
   - Non-200 status
   - GraphQL errors
   - malformed payloads

5. `Observability Failures`
   - Snapshot timeout
   - Dashboard render errors
   - Log sink configuration failure

### 14.2 Recovery Behavior

- Dispatch validation failures:
  - Skip new dispatches.
  - Keep service alive.
  - Continue reconciliation where possible.

- Worker failures:
  - Convert to retries with exponential backoff.

- Tracker candidate-fetch failures:
  - Skip this tick.
  - Try again on next tick.

- Reconciliation state-refresh failures:
  - Keep current workers.
  - Retry on next tick.

- Dashboard/log failures:
  - Do not crash the orchestrator.

### 14.3 Partial State Recovery (Restart)

Current design is intentionally in-memory for scheduler state.
Restart recovery means the service can resume useful operation by polling tracker state and reusing
preserved workspaces. It does not mean retry timers, running sessions, or live worker state survive
process restart.

After restart:

- No retry timers are restored from prior process memory.
- No running sessions are assumed recoverable.
- Service recovers by:
  - startup terminal workspace cleanup
  - working-state reset if `tracker.working_state` is configured (Section 8.9)
  - fresh polling of active issues
  - re-dispatching eligible work

#### 14.3.1 Persistent Budget State Across Restarts (Recommended Extension)

Implementations with budget guards (Section 8.8) should persist per-issue limit counters and
abandoned-issue sets to disk so that caps survive process restarts. Without persistence, a restart
resets all counters to zero, allowing a capped issue to consume another full budget cycle.

Recommended approach:

- Store `issue_limits` (per-issue continuation count, failure retry count, cumulative cost) and
  `abandoned` (set of issue IDs that hit a budget cap) in a JSON file.
- Use atomic writes (write to a temporary file, then rename into place) to avoid corruption on
  crash.
- Use per-workflow file isolation (for example `issue-limits-dev.json`,
  `issue-limits-pr-review.json`) to prevent cross-contamination between parallel orchestrators.
- Load the file at the top of `start()` before the first tick.
- Save after every counter mutation. Burst mutations during a busy tick should be coalesced into one
  deferred write.
- Clear per-issue entries when an issue reaches a terminal state naturally (success or external
  transition detected by reconciliation). Do not clear on abandonment — the whole point is that
  abandonment must survive restart.
- Quarantine corrupt files (for example rename to `*.corrupt-<timestamp>`) on load and start fresh.

### 14.4 Operator Intervention Points

Operators can control behavior by:

- Editing `WORKFLOW.md` (prompt and most runtime settings).
- `WORKFLOW.md` changes are detected and re-applied automatically without restart according to
  Section 6.2.
- Changing issue states in the tracker:
  - terminal state -> running session is stopped and workspace cleaned when reconciled
  - non-active state -> running session is stopped without cleanup
- Restarting the service for process recovery or deployment (not as the normal path for applying
  workflow config changes).

## 15. Security and Operational Safety

### 15.1 Trust Boundary Assumption

Each implementation defines its own trust boundary.

Operational safety requirements:

- Implementations SHOULD state clearly whether they are intended for trusted environments, more
  restrictive environments, or both.
- Implementations SHOULD state clearly whether they rely on auto-approved actions, operator
  approvals, stricter sandboxing, or some combination of those controls.
- Workspace isolation and path validation are important baseline controls, but they are not a
  substitute for whatever approval and sandbox policy an implementation chooses.

### 15.2 Filesystem Safety Requirements

Mandatory:

- Workspace path MUST remain under configured workspace root.
- Coding-agent cwd MUST be the per-issue workspace path for the current run.
- Workspace directory names MUST use sanitized identifiers.

RECOMMENDED additional hardening for ports:

- Run under a dedicated OS user.
- Restrict workspace root permissions.
- Mount workspace root on a dedicated volume if possible.

### 15.3 Secret Handling

- Support `$VAR` indirection in workflow config.
- Do not log API tokens or secret env values.
- Validate presence of secrets without printing them.

### 15.4 Hook Script Safety

Workspace hooks are arbitrary shell scripts from `WORKFLOW.md`.

Implications:

- Hooks are fully trusted configuration.
- Hooks run inside the workspace directory.
- Hook output SHOULD be truncated in logs.
- Hook timeouts are REQUIRED to avoid hanging the orchestrator.

### 15.5 Harness Hardening Guidance

Running Codex agents against repositories, issue trackers, and other inputs that can contain
sensitive data or externally-controlled content can be dangerous. A permissive deployment can lead
to data leaks, destructive mutations, or full machine compromise if the agent is induced to execute
harmful commands or use overly-powerful integrations.

Implementations SHOULD explicitly evaluate their own risk profile and harden the execution harness
where appropriate. This specification intentionally does not mandate a single hardening posture, but
implementations SHOULD NOT assume that tracker data, repository contents, prompt inputs, or tool
arguments are fully trustworthy just because they originate inside a normal workflow.

Possible hardening measures include:

- Tightening Codex approval and sandbox settings described elsewhere in this specification instead
  of running with a maximally permissive configuration.
- Adding external isolation layers such as OS/container/VM sandboxing, network restrictions, or
  separate credentials beyond the built-in Codex policy controls.
- Filtering which Linear issues, projects, teams, labels, or other tracker sources are eligible for
  dispatch so untrusted or out-of-scope tasks do not automatically reach the agent.
- Narrowing the `linear_graphql` tool so it can only read or mutate data inside the
  intended project scope, rather than exposing general workspace-wide tracker access.
- Reducing the set of client-side tools, credentials, filesystem paths, and network destinations
  available to the agent to the minimum needed for the workflow.

The correct controls are deployment-specific, but implementations SHOULD document them clearly and
treat harness hardening as part of the core safety model rather than an optional afterthought.

## 16. Reference Algorithms (Language-Agnostic)

### 16.1 Service Startup

```text
function start_service():
  configure_logging()
  start_observability_outputs()
  start_workflow_watch(on_change=reload_and_reapply_workflow)

  state = {
    poll_interval_ms: get_config_poll_interval_ms(),
    max_concurrent_agents: get_config_max_concurrent_agents(),
    running: {},
    claimed: set(),
    retry_attempts: {},
    completed: set(),
    codex_totals: {input_tokens: 0, output_tokens: 0, total_tokens: 0, seconds_running: 0},
    codex_rate_limits: null
  }

  validation = validate_dispatch_config()
  if validation is not ok:
    log_validation_error(validation)
    fail_startup(validation)

  startup_terminal_workspace_cleanup()
  schedule_tick(delay_ms=0)

  event_loop(state)
```

### 16.2 Poll-and-Dispatch Tick

```text
on_tick(state):
  state = reconcile_running_issues(state)

  validation = validate_dispatch_config()
  if validation is not ok:
    log_validation_error(validation)
    notify_observers()
    schedule_tick(state.poll_interval_ms)
    return state

  issues = tracker.fetch_candidate_issues()
  if issues failed:
    log_tracker_error()
    notify_observers()
    schedule_tick(state.poll_interval_ms)
    return state

  for issue in sort_for_dispatch(issues):
    if no_available_slots(state):
      break

    if should_dispatch(issue, state):
      state = dispatch_issue(issue, state, attempt=null)

  notify_observers()
  schedule_tick(state.poll_interval_ms)
  return state
```

### 16.3 Reconcile Active Runs

```text
function reconcile_running_issues(state):
  state = reconcile_stalled_runs(state)

  running_ids = keys(state.running)
  if running_ids is empty:
    return state

  refreshed = tracker.fetch_issue_states_by_ids(running_ids)
  if refreshed failed:
    log_debug("keep workers running")
    return state

  for issue in refreshed:
    if issue.state in terminal_states:
      state = terminate_running_issue(state, issue.id, cleanup_workspace=true)
    else if issue.state in active_states:
      state.running[issue.id].issue = issue
    else:
      state = terminate_running_issue(state, issue.id, cleanup_workspace=false)

  return state
```

### 16.4 Dispatch One Issue

```text
function dispatch_issue(issue, state, attempt):
  worker = spawn_worker(
    fn -> run_agent_attempt(issue, attempt, parent_orchestrator_pid) end
  )

  if worker spawn failed:
    return schedule_retry(state, issue.id, next_attempt(attempt), {
      identifier: issue.identifier,
      error: "failed to spawn agent"
    })

  state.running[issue.id] = {
    worker_handle,
    monitor_handle,
    identifier: issue.identifier,
    issue,
    session_id: null,
    codex_app_server_pid: null,
    last_codex_message: null,
    last_codex_event: null,
    last_codex_timestamp: null,
    codex_input_tokens: 0,
    codex_output_tokens: 0,
    codex_total_tokens: 0,
    last_reported_input_tokens: 0,
    last_reported_output_tokens: 0,
    last_reported_total_tokens: 0,
    retry_attempt: normalize_attempt(attempt),
    started_at: now_utc()
  }

  state.claimed.add(issue.id)
  state.retry_attempts.remove(issue.id)
  return state
```

### 16.5 Worker Attempt (Workspace + Prompt + Agent)

```text
function run_agent_attempt(issue, attempt, orchestrator_channel):
  workspace = workspace_manager.create_for_issue(issue.identifier)
  if workspace failed:
    fail_worker("workspace error")

  if run_hook("before_run", workspace.path) failed:
    fail_worker("before_run hook error")

  session = app_server.start_session(workspace=workspace.path)
  if session failed:
    run_hook_best_effort("after_run", workspace.path)
    fail_worker("agent session startup error")

  max_turns = config.agent.max_turns
  turn_number = 1

  while true:
    prompt = build_turn_prompt(workflow_template, issue, attempt, turn_number, max_turns)
    if prompt failed:
      app_server.stop_session(session)
      run_hook_best_effort("after_run", workspace.path)
      fail_worker("prompt error")

    turn_result = app_server.run_turn(
      session=session,
      prompt=prompt,
      issue=issue,
      on_message=(msg) -> send(orchestrator_channel, {codex_update, issue.id, msg})
    )

    if turn_result failed:
      app_server.stop_session(session)
      run_hook_best_effort("after_run", workspace.path)
      fail_worker("agent turn error")

    refreshed_issue = tracker.fetch_issue_states_by_ids([issue.id])
    if refreshed_issue failed:
      app_server.stop_session(session)
      run_hook_best_effort("after_run", workspace.path)
      fail_worker("issue state refresh error")

    issue = refreshed_issue[0] or issue

    if issue.state is not active:
      break

    if turn_number >= max_turns:
      break

    turn_number = turn_number + 1

  app_server.stop_session(session)
  run_hook_best_effort("after_run", workspace.path)

  exit_normal()
```

### 16.6 Worker Exit and Retry Handling

```text
on_worker_exit(issue_id, reason, state):
  running_entry = state.running.remove(issue_id)
  state = add_runtime_seconds_to_totals(state, running_entry)

  if reason == normal:
    state.completed.add(issue_id)  # bookkeeping only
    state = schedule_retry(state, issue_id, 1, {
      identifier: running_entry.identifier,
      delay_type: continuation
    })
  else:
    state = schedule_retry(state, issue_id, next_attempt_from(running_entry), {
      identifier: running_entry.identifier,
      error: format("worker exited: %reason")
    })

  notify_observers()
  return state
```

```text
on_retry_timer(issue_id, state):
  retry_entry = state.retry_attempts.pop(issue_id)
  if missing:
    return state

  candidates = tracker.fetch_candidate_issues()
  if fetch failed:
    return schedule_retry(state, issue_id, retry_entry.attempt + 1, {
      identifier: retry_entry.identifier,
      error: "retry poll failed"
    })

  issue = find_by_id(candidates, issue_id)
  if issue is null:
    state.claimed.remove(issue_id)
    return state

  if available_slots(state) == 0:
    return schedule_retry(state, issue_id, retry_entry.attempt + 1, {
      identifier: issue.identifier,
      error: "no available orchestrator slots"
    })

  return dispatch_issue(issue, state, attempt=retry_entry.attempt)
```

## 17. Test and Validation Matrix

A conforming implementation SHOULD include tests that cover the behaviors defined in this
specification.

Validation profiles:

- `Core Conformance`: deterministic tests REQUIRED for all conforming implementations.
- `Extension Conformance`: REQUIRED only for OPTIONAL features that an implementation chooses to
  ship.
- `Real Integration Profile`: environment-dependent smoke/integration checks RECOMMENDED before
  production use.

Unless otherwise noted, Sections 17.1 through 17.7 are `Core Conformance`. Bullets that begin with
`If ... is implemented` are `Extension Conformance`.

### 17.1 Workflow and Config Parsing

- Workflow file path precedence:
  - explicit runtime path is used when provided
  - cwd default is `WORKFLOW.md` when no explicit runtime path is provided
- Workflow file changes are detected and trigger re-read/re-apply without restart
- Invalid workflow reload keeps last known good effective configuration and emits an
  operator-visible error
- Missing `WORKFLOW.md` returns typed error
- Invalid YAML front matter returns typed error
- Front matter non-map returns typed error
- Config defaults apply when OPTIONAL values are missing
- `tracker.kind` validation enforces a supported tracker kind
- `tracker.api_key` works (including `$VAR` indirection)
- `$VAR` resolution works for tracker API key and path values
- `~` path expansion works
- `codex.command` is preserved as a shell command string
- Per-state concurrency override map normalizes state names and ignores invalid values
- Prompt template renders `issue` and `attempt`
- Prompt rendering fails on unknown variables (strict mode)

### 17.2 Workspace Manager and Safety

- Deterministic workspace path per issue identifier
- Missing workspace directory is created
- Existing workspace directory is reused
- Existing non-directory path at workspace location is handled safely (replace or fail per
  implementation policy)
- OPTIONAL workspace population/synchronization errors are surfaced
- `after_create` hook runs only on new workspace creation
- `before_run` hook runs before each attempt and failure/timeouts abort the current attempt
- `after_run` hook runs after each attempt and failure/timeouts are logged and ignored
- `before_remove` hook runs on cleanup and failures/timeouts are ignored
- Workspace path sanitization and root containment invariants are enforced before agent launch
- Agent launch uses the per-issue workspace path as cwd and rejects out-of-root paths

### 17.3 Issue Tracker Client

- Candidate issue fetch uses active states and project slug
- Tracker query uses the tracker-kind-specific project identifier (for example Linear `slugId`,
  GitHub `project_number`)
- Empty `fetch_issues_by_states([])` returns empty without API call
- Pagination preserves order across multiple pages
- Blockers are normalized from inverse relations of type `blocks`
- Labels are normalized to lowercase
- Issue state refresh by ID returns minimal normalized issues
- Issue state refresh query uses the tracker-kind-appropriate ID typing
- Error mapping for request errors, non-200, GraphQL errors, malformed payloads

### 17.4 Orchestrator Dispatch, Reconciliation, and Retry

- Dispatch sort order is priority then oldest creation time
- `Todo` issue with non-terminal blockers is not eligible
- `Todo` issue with terminal blockers is eligible
- Active-state issue refresh updates running entry state
- Non-active state stops running agent without workspace cleanup
- Terminal state stops running agent and cleans workspace
- Reconciliation with no running issues is a no-op
- Normal worker exit schedules a short continuation retry (attempt 1)
- Abnormal worker exit increments retries with 10s-based exponential backoff
- Retry backoff cap uses configured `agent.max_retry_backoff_ms`
- Retry queue entries include attempt, due time, identifier, and error
- Stall detection kills stalled sessions and schedules retry
- Slot exhaustion requeues retries with explicit error reason
- If a snapshot API is implemented, it returns running rows, retry rows, token totals, and rate
  limits
- If a snapshot API is implemented, timeout/unavailable cases are surfaced

### 17.5 Coding-Agent Client

- Launch command uses workspace cwd and invokes the agent backend startup sequence (for example
  `bash -lc <codex.command>` for Codex, or direct CLI spawn for Claude Code)
- Session startup follows the targeted agent backend's protocol (for example `initialize` /
  `initialized` / `thread/start` / `turn/start` for Codex app-server; implicit start for CLI-based
  agents)
- Client identity/capability payloads are valid when the targeted agent protocol requires them
- Policy-related startup payloads use the implementation's documented approval/sandbox settings
- Thread and turn identities exposed by the targeted protocol are extracted and used to emit
  `session_started`
- Request/response read timeout is enforced
- Turn timeout is enforced
- Transport framing required by the targeted protocol is handled correctly
- For stdio-based transports, diagnostic stderr handling is kept separate from the protocol stream
- Command/file-change approvals are handled according to the implementation's documented policy
- Unsupported dynamic tool calls are rejected without stalling the session
- User input requests are handled according to the implementation's documented policy and do not
  stall indefinitely
- Usage and rate-limit telemetry exposed by the targeted protocol is extracted
- Approval, user-input-required, usage, and rate-limit signals are interpreted according to the
  targeted protocol
- If client-side tools are implemented, session startup advertises the supported tool specs
  using the targeted app-server protocol
- If the `linear_graphql` client-side tool extension is implemented:
  - the tool is advertised to the session
  - valid `query` / `variables` inputs execute against configured Linear auth
  - top-level GraphQL `errors` produce `success=false` while preserving the GraphQL body
  - invalid arguments, missing auth, and transport failures return structured failure payloads
  - unsupported tool names still fail without stalling the session

### 17.6 Observability

- Validation failures are operator-visible
- Structured logging includes issue/session context fields
- Logging sink failures do not crash orchestration
- Token/rate-limit aggregation remains correct across repeated agent updates
- If a human-readable status surface is implemented, it is driven from orchestrator state and does
  not affect correctness
- If humanized event summaries are implemented, they cover key wrapper/agent event classes without
  changing orchestrator behavior

### 17.7 CLI and Host Lifecycle

- CLI accepts a positional workflow path argument (`path-to-WORKFLOW.md`)
- CLI uses `./WORKFLOW.md` when no workflow path argument is provided
- CLI errors on nonexistent explicit workflow path or missing default `./WORKFLOW.md`
- CLI surfaces startup failure cleanly
- CLI exits with success when application starts and shuts down normally
- CLI exits nonzero when startup fails or the host process exits abnormally

### 17.8 Real Integration Profile (RECOMMENDED)

These checks are RECOMMENDED for production readiness and MAY be skipped in CI when credentials,
network access, or external service permissions are unavailable.

- A real tracker smoke test can be run with valid credentials supplied by `LINEAR_API_KEY` or a
  documented local bootstrap mechanism (for example `~/.linear_api_key`).
- Real integration tests SHOULD use isolated test identifiers/workspaces and clean up tracker
  artifacts when practical.
- A skipped real-integration test SHOULD be reported as skipped, not silently treated as passed.
- If a real-integration profile is explicitly enabled in CI or release validation, failures SHOULD
  fail that job.

## 18. Implementation Checklist (Definition of Done)

Use the same validation profiles as Section 17:

- Section 18.1 = `Core Conformance`
- Section 18.2 = `Extension Conformance`
- Section 18.3 = `Real Integration Profile`

### 18.1 REQUIRED for Conformance

- Workflow path selection supports explicit runtime path and cwd default
- `WORKFLOW.md` loader with YAML front matter + prompt body split
- Typed config layer with defaults and `$` resolution
- Dynamic `WORKFLOW.md` watch/reload/re-apply for config and prompt
- Polling orchestrator with single-authority mutable state
- Issue tracker client with candidate fetch + state refresh + terminal fetch
- Workspace manager with sanitized per-issue workspaces
- Workspace lifecycle hooks (`after_create`, `before_run`, `after_run`, `before_remove`)
- Hook timeout config (`hooks.timeout_ms`, default `60000`)
- Coding-agent app-server subprocess client with JSON line protocol
- Codex launch command config (`codex.command`, default `codex app-server`)
- Strict prompt rendering with `issue` and `attempt` variables
- Exponential retry queue with continuation retries after normal exit
- Configurable retry backoff cap (`agent.max_retry_backoff_ms`, default 5m)
- Reconciliation that stops runs on terminal/non-active tracker states
- Workspace cleanup for terminal issues (startup sweep + active transition)
- Structured logs with `issue_id`, `issue_identifier`, and `session_id`
- Operator-visible observability (structured logs; OPTIONAL snapshot/status surface)

### 18.2 RECOMMENDED Extensions (Not REQUIRED for Conformance)

- HTTP server extension honors CLI `--port` over `server.port`, uses a safe default bind host, and
  exposes the baseline endpoints/error semantics in Section 13.7 if shipped.
- `linear_graphql` client-side tool extension exposes raw Linear GraphQL access through the
  app-server session using configured Symphony auth.
- Multiple workflow files per codebase with a registry mapping deployments to their config (Section
  5.1.1).
- Closed-alphabet `verdict_state_map` routing with explicit `done_state` and
  `paused_states`/`backlog_states` categories (Sections 5.3.1, 5.3.7).
- Per-workflow `rules` block declaring worker `must`/`must_not`/`limits` separately from prompt
  body (Section 5.3.8).
- Generated workflow alphabets feeding compile-time exhaustiveness checks (Section 5.6).
- Pure reducer / interpreter split (TEA architecture) with mechanical purity enforcement (Section
  7.5).
- Rate-limit gating that suppresses new dispatch when the agent emits rate-limit events with timing
  hints (Section 8.7).
- Per-issue budget guards: `max_continuations`, `max_failure_retries`, `max_cost_per_issue_usd`
  (Section 8.8).
- Working-state dispatch lock with startup reset for crash recovery (Section 8.9).
- Operator halt / andon cord with closed-union halt categories (Sections 8.10, 8.10.1).
- Fleet-wide token-based budget ledger spanning multiple orchestrator processes (Section 8.11).
- Pre-dispatch guard chain (abandoned-desync, loop detector, etc.) (Section 8.12).
- Workspace preservation on abandonment via `abandoned/<branch>-<ts>` push instead of `git reset`
  (Section 9.6).
- Tick watchdog plus external supervisor (launchd/systemd) with process-group reaper (Section 9.7).
- Orchestrator-driven tracker writes for audit trail and distributed coordination (Section 11.6).
- Optimistic concurrency / epoch CAS on tracker writes (Section 11.6.1).
- Pluggable issue tracker adapters beyond Linear, for example GitHub Projects v2 (Section 11.2b).
- SSE streaming endpoints for real-time dashboard updates (Section 13.7.3).
- Per-session NDJSON structured logging (Section 13.8).
- Reducer event log (append-only NDJSON of every event + emitted effects) (Section 13.9).
- Operator intervention NDJSON log distinct from worker-driven transitions (Section 13.10).
- Persistent budget state across restarts (Section 14.3.1).

### 18.3 Operational Validation Before Production (RECOMMENDED)

- Run the `Real Integration Profile` from Section 17.8 with valid credentials and network access.
- Verify hook execution and workflow path resolution on the target host OS/shell environment.
- If the OPTIONAL HTTP server is shipped, verify the configured port behavior and loopback/default
  bind expectations on the target environment.

## Appendix A. SSH Worker Extension (OPTIONAL)

This appendix describes a common extension profile in which Symphony keeps one central
orchestrator but executes worker runs on one or more remote hosts over SSH.

Extension config:

- `worker.ssh_hosts` (list of SSH host strings, OPTIONAL)
  - When omitted, work runs locally.
- `worker.max_concurrent_agents_per_host` (positive integer, OPTIONAL)
  - Shared per-host cap applied across configured SSH hosts.

### A.1 Execution Model

- The orchestrator remains the single source of truth for polling, claims, retries, and
  reconciliation.
- `worker.ssh_hosts` provides the candidate SSH destinations for remote execution.
- Each worker run is assigned to one host at a time, and that host becomes part of the run's
  effective execution identity along with the issue workspace.
- `workspace.root` is interpreted on the remote host, not on the orchestrator host.
- The coding-agent app-server is launched over SSH stdio instead of as a local subprocess, so the
  orchestrator still owns the session lifecycle even though commands execute remotely.
- Continuation turns inside one worker lifetime SHOULD stay on the same host and workspace.
- A remote host SHOULD satisfy the same basic contract as a local worker environment: reachable
  shell, writable workspace root, coding-agent executable, and any required auth or repository
  prerequisites.

### A.2 Scheduling Notes

- SSH hosts MAY be treated as a pool for dispatch.
- Implementations MAY prefer the previously used host on retries when that host is still
  available.
- `worker.max_concurrent_agents_per_host` is an OPTIONAL shared per-host cap across configured SSH
  hosts.
- When all SSH hosts are at capacity, dispatch SHOULD wait rather than silently falling back to a
  different execution mode.
- Implementations MAY fail over to another host when the original host is unavailable before work
  has meaningfully started.
- Once a run has already produced side effects, a transparent rerun on another host SHOULD be
  treated as a new attempt, not as invisible failover.

### A.3 Problems to Consider

- Remote environment drift:
  - Each host needs the expected shell environment, coding-agent executable, auth, and repository
    prerequisites.
- Workspace locality:
  - Workspaces are usually host-local, so moving an issue to a different host is typically a cold
    restart unless shared storage exists.
- Path and command safety:
  - Remote path resolution, shell quoting, and workspace-boundary checks matter more once execution
    crosses a machine boundary.
- Startup and failover semantics:
  - Implementations SHOULD distinguish host-connectivity/startup failures from in-workspace agent
    failures so the same ticket is not accidentally re-executed on multiple hosts.
- Host health and saturation:
  - A dead or overloaded host SHOULD reduce available capacity, not cause duplicate execution or an
    accidental fallback to local work.
- Cleanup and observability:
  - Operators need to know which host owns a run, where its workspace lives, and whether cleanup
    happened on the right machine.

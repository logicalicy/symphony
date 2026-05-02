# Symphony

Symphony turns project work into isolated, autonomous implementation runs, allowing teams to manage
work instead of supervising coding agents.

[![Symphony demo video preview](.github/media/symphony-demo-poster.jpg)](.github/media/symphony-demo.mp4)

_In this [demo video](.github/media/symphony-demo.mp4), Symphony monitors a Linear board for work and spawns agents to handle the tasks. The agents complete the tasks and provide proof of work: CI status, PR review feedback, complexity analysis, and walkthrough videos. When accepted, the agents land the PR safely. Engineers do not need to supervise Codex; they can manage the work at a higher level._

> [!WARNING]
> Symphony is a low-key engineering preview for testing in trusted environments.

## Running Symphony

### Requirements

Symphony works best in codebases that have adopted
[harness engineering](https://openai.com/index/harness-engineering/). Symphony is the next step --
moving from managing coding agents to managing work that needs to get done.

### Option 1. Make your own

Tell your favorite coding agent to build Symphony in a programming language of your choice:

> Implement Symphony according to the following spec:
> https://github.com/openai/symphony/blob/main/SPEC.md

### Option 2. Use our experimental reference implementation

Check out [elixir/README.md](elixir/README.md) for instructions on how to set up your environment
and run the Elixir-based Symphony implementation. You can also ask your favorite coding agent to
help with the setup:

> Set up Symphony for my repository based on
> https://github.com/openai/symphony/blob/main/elixir/README.md

---

## Known Implementations

### Elixir (Reference)

The [elixir/](elixir/) directory contains the reference implementation using Elixir/OTP, Linear, and
Codex app-server. See [elixir/README.md](elixir/README.md) for setup.

### TypeScript + Claude Code + GitHub

A production TypeScript implementation using Claude Code CLI and GitHub Projects v2 instead of Codex
and Linear.

| Aspect | Upstream Spec | This Implementation |
|--------|--------------|---------------------|
| Language | Language-agnostic | TypeScript |
| Tracker | Linear | GitHub Projects v2 (`tracker.kind: github`) |
| Agent | Codex app-server (JSON-RPC) | Claude Code CLI (`--output-format stream-json`) |
| Config section | `codex:` | `claude:` |
| Continuation | `turn/start` on live thread | `--resume <sessionId>` subprocess respawn |

Key extensions implemented beyond the core spec:

- **Pure reducer architecture** — TEA-style split with mechanical purity enforcement (Section 7.5)
- **Closed verdict alphabet + state map** — explicit `verdict_state_map` routes worker tokens to
  board states (Sections 5.3.1, 5.3.7)
- **Per-workflow rules block** — `must`/`must_not`/`limits` declared separately from prompt body
  (Section 5.3.8)
- **Generated workflow alphabets** — codegen drives compile-time exhaustiveness checks (Section
  5.6)
- **Budget guards** — `max_continuations`, `max_failure_retries`, `max_cost_per_issue_usd` (Section
  8.8)
- **Persistent budget state** — atomic JSON file persistence across restarts (Section 14.3.1)
- **Rate-limit gating** — suppresses dispatch on 429 events with timing hints (Section 8.7)
- **Working-state dispatch lock** — tracker-visible distributed lock with startup reset (Section
  8.9)
- **Operator halt / andon cord** — closed-union halt categories, filesystem lock, no programmatic
  clear (Sections 8.10, 8.10.1)
- **Fleet-wide token budget** — token-based ledger (not USD) shared across orchestrator processes
  via flock (Section 8.11)
- **Pre-dispatch guard chain** — abandoned-desync, loop detector, etc. (Section 8.12)
- **Workspace preservation on abandonment** — `abandoned/<branch>-<ts>` push instead of
  `git reset` (Section 9.6)
- **Tick watchdog + external supervisor** — launchd respawn + `pkill -TERM -- -$PGID` reaper
  (Section 9.7)
- **Orchestrator tracker writes** — audit comments and state transitions (Section 11.6)
- **Epoch CAS on tracker writes** — optimistic concurrency via `symphony:epoch:N` labels (Section
  11.6.1)
- **SSE streaming dashboard** — real-time event streaming alongside REST API (Section 13.7.3)
- **Per-session NDJSON logging** — one structured log file per issue per attempt (Section 13.8)
- **Reducer event log** — append-only NDJSON of every event + emitted effects (Section 13.9)
- **Operator intervention NDJSON** — human overrides logged separately from worker transitions
  (Section 13.10)
- **Multiple workflow files** — parallel orchestrators via worker registry (Section 5.1.1)

---

## License

This project is licensed under the [Apache License 2.0](LICENSE).

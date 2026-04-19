# session-manager

LLM-agnostic session orchestrator that coordinates concurrent AI sessions — Claude Code, Codex, Gemini CLI, Aider, or any future tool. Entity-tag overlap detection, conflict awareness, relevance-filtered context injection, and a durable work journal. Zero operator input at launch.

## The Problem

LLM sessions are isolated. Each starts with zero knowledge of other active sessions — what they're working on, what files they've touched, what decisions they've made. The operator is the only bridge, manually relaying context between sessions. This doesn't scale past two concurrent sessions.

Three problems compound:

- **Context loss** — starting a new session requires re-explaining what other sessions have done
- **Blind conflicts** — two sessions can modify the same file without either knowing (caught by Git post-hoc, not prevented)
- **No fleet visibility** — no way to see how many sessions are active, what each is doing, or which files are under contention

## How It Works

A single privileged process — the orchestrator — owns fleet coordination, narration, and context routing. Sessions emit structured events; the orchestrator decides what's relevant and injects context proportional to overlap.

**Relevance, not broadcast.** Sessions receive context only about other sessions whose entity tags overlap with their own. Zero overlap = zero injection = zero tokens spent.

**Filesystem is the coordination layer.** No IPC, no sockets, no network. Sessions communicate via shared files that any LLM tool can read regardless of runtime.

**Token cost awareness.** Orchestration mechanics (heartbeat checking, overlap computation, conflict detection) run as compiled Rust — zero LLM tokens. Only narration synthesis uses an LLM, routed to the cheapest capable model. Frontier models stay in the sessions doing actual work.

## Architecture

Three layers, cleanly separated:

| Layer | Implementation | Token Cost |
|---|---|---|
| **Sessions** | Thin shim wrapping any LLM CLI (captures PID, writes heartbeats) | 0 |
| **Orchestrator core** | Rust binary — heartbeat monitoring, overlap computation, conflict detection, registry consolidation | 0 |
| **Narration engine** | Haiku API — clusters events into summaries every 30-60s | ~200-500/call |

### Filesystem layout

All session state lives in a shared directory:

```
sessions/
├── events.jsonl              # append-only event bus (all sessions write)
├── registry.json             # consolidated registry (orchestrator writes)
├── entities.toml             # auto-learned entity graph
├── heartbeats/
│   └── {session-id}.json     # per-session heartbeat (shim writes)
└── context/
    └── {session-id}.json     # per-session context injection (orchestrator writes)
```

## Entity Tags and Overlap

Tags emerge during a session's lifetime from observed activity — zero operator input:

- **Files read/written** — project inferred from path, language from extension
- **SSH connections** — host entity inferred from hostname
- **Git activity** — repo/project name from `.git/logs/HEAD` mtime
- **Process children** — tools invoked (cargo → Rust, npm → TypeScript)

The orchestrator re-computes overlap every time a tag set changes.

### Overlap-driven injection tiers

| Overlap Depth | Injection | Token Cost |
|---|---|---|
| 0 tags | Nothing | 0 |
| 1 tag (indirect) | One-liner summary | ~30 |
| 2+ tags (direct) | Task + recent action | ~100 |
| Same file modified | Conflict alert | ~150 |

## Conflict Detection

When the orchestrator consolidates heartbeats, it checks for overlapping `files_modified`. If two sessions list the same file, the relevant context files are annotated with a warning.

The orchestrator detects conflicts but does not resolve them — that's the operator's decision.

## Work Journal

A durable, queryable record of what each session has done, decided, and deferred. Six entry types: **achievement**, **decision**, **todo**, **aspiration**, **learning**, **blocker**.

Entries are project-scoped JSONL — append-only, one per line. Three creation modes: explicit (operator invokes), inferred (from git commits and logs), hooked (session-end summary).

## Design Principles

- **LLM-agnostic** — no hooks, plugins, or extensions required. Any CLI that runs in a terminal can participate
- **Relevance, not broadcast** — zero overlap = zero context injected = zero tokens spent
- **Filesystem coordination** — no IPC, no sockets. All communication via shared files
- **Graceful degradation** — if the orchestrator isn't running, sessions work in isolation as they do today
- **Token cost awareness** — ~70% of orchestration is deterministic compiled code
- **High-precision timestamps** — ISO-8601 with nanosecond precision for deterministic ordering

## Roadmap

### Phase 1 — Core Orchestrator

- [ ] Rust binary: heartbeat monitor, registry consolidation
- [ ] Session shim wrapper for LLM CLIs
- [ ] Entity-tag inference from PID tree observation
- [ ] Overlap computation and context injection

### Phase 2 — Conflict Detection and Journal

- [ ] File-level conflict detection across sessions
- [ ] Work journal storage (JSONL) and session-end hooks
- [ ] Journal context injection on session start

### Phase 3 — Narration and Fleet Visibility

- [ ] Narration engine integration (Haiku API)
- [ ] Voice endpoint integration (bearer-token protected)
- [ ] Operator CLI for fleet status and session inspection

### Phase 4 — Entity Graph Learning

- [ ] Auto-learned entity correlations with confidence thresholds
- [ ] Cross-project relationship inference
- [ ] Entity graph persistence and self-healing rebuild

## Current Status

**Phase:** Design complete — spec published, implementation pending.

The session manager is fully specified with architecture, entity-tag model, overlap computation, conflict detection, work journal, and implementation priorities defined. The specification is the deliverable for this phase.

## Language

Rust

## License

Apache 2.0

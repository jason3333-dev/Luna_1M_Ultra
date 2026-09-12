# Codex Astra + Luna 1M

A lightweight Codex multi-agent configuration built around two models only:

- **Astra** — orchestration, hard technical decisions, integration, final acceptance
- **Luna** — exploration, implementation, debugging, testing, and review

The setup avoids Sol/Terra handoff layers and instead reuses Luna context where it is useful.

## Why this setup

A common multi-agent pattern adds multiple model tiers between orchestration and implementation. That can duplicate context, repeat repository exploration, and add handoff overhead.

This configuration keeps the hierarchy simple:

```text
Astra max
├─ Luna low      → search / lookup
├─ Luna medium   → analysis
├─ Luna high     → small fixes / normal review
└─ Luna max      → implementation / difficult debugging / deep review
```

Luna is inexpensive enough that retaining useful working context can be more efficient than repeatedly rebuilding it. Astra stays focused on decisions that benefit from already having the main-session context.

## Features

- Astra as the single top-level orchestrator
- Luna-only delegated agents
- Task-based Luna reasoning levels: `low`, `medium`, `high`, `max`
- 1M requested context window for Astra and Luna roles
- 900K starting point for auto-compaction
- Context reuse across investigation → implementation → testing → revision
- No mandatory reasoning ladder
- No Sol/Terra escalation layer
- Separate read-only review roles

## Setup

See [Codex_Astra_Luna_1M_Setup.md](./Codex_Astra_Luna_1M_Setup.md).

At a minimum, the setup adds:

```toml
model = "gpt-6-astra"
model_reasoning_effort = "max"
model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000

[agents]
enabled = true
default_subagent_model = "gpt-5.6-luna"
default_subagent_reasoning_effort = "medium"
max_concurrent_threads_per_session = 12
```

and defines Luna roles under:

```text
~/.codex/agents/
├── luna_scan.toml
├── luna_analyze.toml
├── luna_quickfix.toml
├── luna_max.toml
├── luna_review.toml
└── luna_review_max.toml
```

## Role selection

| Role | Effort | Use case |
|---|---:|---|
| `luna_scan` | low | file/symbol lookup, exact searches |
| `luna_analyze` | medium | flow tracing, logs, dependencies |
| `luna_quickfix` | high | small fixes, routine tests |
| `luna_max` | max | substantive implementation, hard debugging |
| `luna_review` | high | normal independent review |
| `luna_review_max` | max | concurrency/state/recovery/invariant review |

These are **task classes, not sequential stages**.

## Context policy

- Reuse an existing Luna worker for related work.
- Start fresh for unrelated one-off tasks.
- Do not force an empty context just to save inexpensive Luna input tokens.
- Do not fill the 1M window simply because it exists.
- If Luna reaches a genuinely hard unresolved decision, send the evidence back to Astra instead of adding another manager model.

## Compatibility note

`model_context_window = 1_000_000` is a requested value. The effective context window can still be limited by the installed Codex build, model catalog, account availability, or project/profile overrides.

## License

MIT License. See [`LICENSE`](./LICENSE).

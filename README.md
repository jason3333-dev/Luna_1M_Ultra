# Codex Astra + Luna Model-Max Context

A lightweight Codex multi-agent configuration built around two models only:

- **Astra** — orchestration, hard technical decisions, integration, final acceptance
- **Luna** — exploration, implementation, debugging, testing, and review

The setup avoids Sol/Terra handoff layers and instead reuses Luna context where it is useful.

## Why this setup

A common multi-agent pattern adds multiple model tiers between orchestration and implementation. That can duplicate context, repeat repository exploration, and add handoff overhead.

This configuration keeps the hierarchy simple:

```text
Astra main/native (global default)
├─ Luna Low      → search / lookup
├─ Luna Medium   → analysis
├─ Luna High     → small fixes / routine tests
├─ Luna Max      → implementation / difficult debugging
└─ Luna Review   → independent review
```

Luna is inexpensive enough that retaining useful working context can be more efficient than repeatedly rebuilding it. Astra stays focused on decisions that benefit from already having the main-session context.

## Features

- Astra as the single top-level orchestrator
- Luna-only delegated agents
- Global default request of `model_context_window = 1_000_000` and `model_auto_compact_token_limit = 900_000`
- Five task-based Luna roles inheriting the global context and compaction default: `low`, `medium`, `high`, `max`, and `review`
- The local model catalog may clamp the request to `max_context_window = 872_000`, with an effective runtime compaction limit of `828_400`
- Context reuse across investigation → implementation → testing → revision
- No mandatory reasoning ladder
- No Sol/Terra escalation layer
- Separate read-only review roles

## Setup

See [Codex_Astra_Luna_1M_Setup.md](./Codex_Astra_Luna_1M_Setup.md).

At a minimum, the setup adds:

```toml
model = "gpt-6-astra"
model_reasoning_effort = "low"
plan_mode_reasoning_effort = "low"
review_model = "gpt-5.6-luna"
model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000

[agents]
enabled = true
default_subagent_model = "gpt-5.6-luna"
default_subagent_reasoning_effort = "medium"
max_concurrent_threads_per_session = 12
```

The five role files under `~/.codex/agents/` inherit the global context and compaction settings above; they do not override either setting. The setup also defines Luna roles under:

```text
~/.codex/agents/
├── luna_low.toml
├── luna_medium.toml
├── luna_high.toml
├── luna_max.toml
├── luna_review.toml
```

## Role selection

| Role | Effort | Context | Use case |
|---|---:|---:|---|
| `luna_low` | low | Global default | file/symbol lookup, exact searches |
| `luna_medium` | medium | Global default | flow tracing, logs, dependencies |
| `luna_high` | high | Global default | small fixes, routine tests |
| `luna_max` | max | Global default | substantive implementation, hard debugging |
| `luna_review` | high | Global default | independent review |

These are **task classes, not sequential stages**.

## Context policy

- Reuse an existing Luna worker for related work.
- Start fresh for unrelated one-off tasks.
- Do not force an empty context just to save inexpensive Luna input tokens.
- Do not fill the global default context window simply because it exists.
- If Luna reaches a genuinely hard unresolved decision, send the evidence back to Astra instead of adding another manager model.

## GitHub routing

- Important GitHub work is explicitly delegated to an existing suitable Luna worker before broad exploration, implementation, debugging, or testing; reuse that worker when possible.
- Luna/max is the default: `luna_max` handles repository search, diff analysis, code or documentation changes, tests, GitHub CLI preparation, and issue/PR drafts; `luna_review` handles independent review.
- Because Luna is low-cost, use a small number of parallel Luna workers proactively when independent work units improve turnaround or coverage; keep their scopes separate.
- Astra is limited to task framing, final scope and safety approval, and public repository creation, push, merge, or permission execution.
- While Luna works, Astra does not repeat the same scope; independent work units may be delegated to Luna in parallel.
- Never publish secrets, local configuration, credentials, or an unreviewed backlog.

## Compatibility note

The global Codex configuration requests `model_context_window = 1_000_000` and `model_auto_compact_token_limit = 900_000` for Astra and all five canonical Luna roles. The current local model catalog may clamp that request to `max_context_window = 872_000` with an effective runtime compaction limit of `828_400`. Role files do not override these settings.

## License

MIT License. See [`LICENSE`](./LICENSE).

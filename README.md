# Codex Astra + Luna Model-Max Context

A lightweight Codex multi-agent configuration built around two models only:

- **Astra** — orchestration, hard technical decisions, integration, final acceptance
- **Luna** — exploration, implementation, debugging, testing, and review

The setup avoids Sol/Terra handoff layers and instead reuses Luna context where it is useful.

## Why this setup

A common multi-agent pattern adds multiple model tiers between orchestration and implementation. That can duplicate context, repeat repository exploration, and add handoff overhead.

This configuration keeps the hierarchy simple:

```text
Astra main/native (~258K)
├─ Luna Low      Codex max → search / lookup
├─ Luna Medium   Codex max → analysis
├─ Luna High     Codex max → small fixes / routine tests
├─ Luna Max      Codex max → implementation / difficult debugging
└─ Luna Review   Codex max → independent review
```

Luna is inexpensive enough that retaining useful working context can be more efficient than repeatedly rebuilding it. Astra stays focused on decisions that benefit from already having the main-session context.

## Features

- Astra as the single top-level orchestrator
- Luna-only delegated agents
- Astra native-default context of approximately 258K
- Five task-based Luna roles at the current Codex model maximum: `low`, `medium`, `high`, `max`, and `review`
- Codex-default context and compaction for Astra; catalog-maximum context and compaction for Luna roles
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

[agents]
enabled = true
default_subagent_model = "gpt-5.6-luna"
default_subagent_reasoning_effort = "medium"
max_concurrent_threads_per_session = 12
```

and defines Luna roles under:

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
| `luna_low` | low | Codex max | file/symbol lookup, exact searches |
| `luna_medium` | medium | Codex max | flow tracing, logs, dependencies |
| `luna_high` | high | Codex max | small fixes, routine tests |
| `luna_max` | max | Codex max | substantive implementation, hard debugging |
| `luna_review` | high | Codex max | independent review |

These are **task classes, not sequential stages**.

## Context policy

- Reuse an existing Luna worker for related work.
- Start fresh for unrelated one-off tasks.
- Do not force an empty context just to save inexpensive Luna input tokens.
- Do not fill the native-default Astra window or a Luna model-maximum window simply because it exists.
- If Luna reaches a genuinely hard unresolved decision, send the evidence back to Astra instead of adding another manager model.

## GitHub routing

- Important GitHub work is explicitly delegated to an existing suitable Luna worker before broad exploration, implementation, debugging, or testing; reuse that worker when possible.
- Luna/max is the default: `luna_max` handles repository search, diff analysis, code or documentation changes, tests, GitHub CLI preparation, and issue/PR drafts; `luna_review` handles independent review.
- Because Luna is low-cost, use a small number of parallel Luna workers proactively when independent work units improve turnaround or coverage; keep their scopes separate.
- Astra is limited to task framing, final scope and safety approval, and public repository creation, push, merge, or permission execution.
- While Luna works, Astra does not repeat the same scope; independent work units may be delegated to Luna in parallel.
- Never publish secrets, local configuration, credentials, or an unreviewed backlog.

## Compatibility note

Luna role files request the maximum context allowed by the current Codex model catalog; Astra leaves context and compaction at Codex defaults (approximately 258K native context in this setup). Effective limits can still be lower because of the installed Codex build, account availability, or project/profile overrides.

## License

MIT License. See [`LICENSE`](./LICENSE).

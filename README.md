# Luna_1M_Ultra

A lightweight Codex multi-agent configuration and prompt/routing profile built around Luna Max as the primary model:

- **Luna Max** — the `gpt-6-luna` default primary session with max reasoning for orchestration, implementation, debugging, testing, and review
- **Astra** — an optional `gpt-6-astra` escalation for explicitly selected hard decisions or final acceptance; it is not the default

`Luna_1M_Ultra` (or “Luna Ultra”) is a conceptual profile name for the prompt, role, context, and routing behavior described here. It is not a Codex model ID or an official model offering. Use the actual model ID `gpt-6-luna` in Luna `model` fields. The published GPT-6 Luna reasoning efforts are `none`, `low`, `medium`, `high`, `xhigh`, and `max`; it does not list `ultra`. Keep `ultra` in the desktop availability list only for other models that support it. See the [official GPT-6 Luna model documentation](https://developers.openai.com/api/docs/models/gpt-6-luna).

This repository is an independent public clone of the profile documentation, not a same-account Git fork.

The profile keeps Luna Max as the primary session and default worker, uses Luna workers as leaves, avoids Sol/Terra handoff layers and Astra child sessions, and reuses Luna context where it is useful.

## Why this setup

A common multi-agent pattern adds multiple model tiers between orchestration and implementation. That can duplicate context, repeat repository exploration, and add handoff overhead.

This configuration keeps the hierarchy simple:

```text
Luna Max primary (`gpt-6-luna`; max reasoning)
├─ Luna Low (`gpt-6-luna`)      → search / lookup
├─ Luna Medium (`gpt-6-luna`)   → analysis
├─ Luna High (`gpt-6-luna`)     → small fixes / routine tests
├─ Luna Max (`gpt-6-luna`)      → implementation / difficult debugging
└─ Luna Review (`gpt-6-luna`)   → independent review
```

Luna is inexpensive enough that retaining useful working context can be more efficient than repeatedly rebuilding it. Luna Max stays focused on decisions that benefit from the primary-session context.

## Features

- Luna Max as the single top-level orchestrator
- Luna-only delegated leaf agents
- Astra as an optional, explicitly selected escalation rather than the default primary model
- No Sol/Terra workers and no Astra child sessions
- `Luna_1M_Ultra` as a conceptual prompt/routing profile, not a model ID
- Global default request of `model_context_window = 1_000_000` and `model_auto_compact_token_limit = 900_000`
- Five task-based Luna roles inheriting the global context and compaction default: `low`, `medium`, `high`, `max`, and `review`
- The published GPT-6 Luna context window is 1.05M tokens; this profile requests 1M, while the active Codex runtime may expose a smaller context window
- Context reuse across investigation → implementation → testing → revision
- No mandatory reasoning ladder
- No Sol/Terra escalation layer
- Separate read-only review roles

## Setup

See [Luna_1M_Ultra_Setup.md](./Luna_1M_Ultra_Setup.md). Existing GPT-5.6 installs can follow the migration notes in that guide.

At a minimum, the setup adds:

```toml
model = "gpt-6-luna"
model_reasoning_effort = "max"
plan_mode_reasoning_effort = "max"
review_model = "gpt-6-luna"
model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000

[desktop]
followUpQueueMode = "stack"
enabled-reasoning-efforts = ["low", "medium", "high", "xhigh", "persistent", "max", "ultra"]

[tui]
status_line = ["model-with-reasoning", "context-remaining", "current-dir"]

[agents]
enabled = true
default_subagent_model = "gpt-6-luna"
default_subagent_reasoning_effort = "max"
max_concurrent_threads_per_session = 12
max_depth = 1

[features.multi_agent_v2]
expose_spawn_agent_model_overrides = true
```

The five role files under `~/.codex/agents/` inherit the global context and compaction settings above; they do not override either setting. Each role uses the actual model ID `gpt-6-luna`; the `Luna_1M_Ultra` profile name must not replace it. The setup also defines Luna roles under:

```text
~/.codex/agents/
├── luna_low.toml
├── luna_medium.toml
├── luna_high.toml
├── luna_max.toml
├── luna_review.toml
```

`expose_spawn_agent_model_overrides = true` makes `spawn_agent` model override controls visible to the orchestrator; it exposes configuration choices, not hidden chain-of-thought.

The desktop reasoning-effort availability list includes `max`. This enables the maximum reasoning option in the model picker; `max` is a reasoning effort, not a separate model ID. The model itself remains `gpt-6-luna`.

The same availability list can include `ultra` for other models that support it. GPT-6 Luna does not list `ultra` as a supported reasoning effort, so the canonical Luna roles stay at `max`.

The TUI footer displays the active model/reasoning, remaining context, and current directory. The `context-remaining` item enables the context-usage UI.

New follow-up prompts use desktop `stack` mode by default, so additional prompt input is queued in order.

## Role selection

| Role | Effort | Context | Use case |
|---|---:|---:|---|
| `luna_low` | low | Global default | file/symbol lookup, exact searches |
| `luna_medium` | medium | Global default | flow tracing, logs, dependencies |
| `luna_high` | high | Global default | small fixes, routine tests |
| `luna_max` | max | Global default | substantive implementation, hard debugging |
| `luna_review` | high | Global default | independent review |

These are **task classes, not sequential stages**.

For each task, dispatch independent investigations, test suites, and reviews concurrently to separate Luna workers. Run dependent work sequentially: wait for the prerequisite result before dispatching the next step.
The default delegated role is `luna_max` with max reasoning; use a lower-effort role only when the task explicitly warrants it.

## Parallel execution use cases

The attached run snapshot shows six independent work units active concurrently. This is an example of task decomposition, not six predefined agent types and not a request to change the model configuration.

![Six parallel Luna Max work units](./parallel-luna-max-run.png)

| Work unit | Use case | Observed duration |
|---|---|---:|
| Core gate | Planning-focused test with a temporary vendor symlink | 2m 48s |
| Operator review | Analyze race conditions between halt and resume | 6m 17s |
| Fill execution p1 | Review canonical delivery identity logic | 11m 11s |
| Accounting p1 | Identify a potential filename-parsing bug | 6m 44s |
| Pair machine p1 | Analyze the impact of an external increase | 11m 46s |
| Candidate p1 | Assess parent-communication options | 10m 58s |

The supplied Pro 5X observation is approximately 1% of the weekly usage window per five minutes during this kind of burst. Treat it as an observed planning reference, not an official quota, billing promise, or guaranteed rate.

## Context policy

- Reuse an existing Luna worker for related work.
- Start fresh for unrelated one-off tasks.
- Do not force an empty context just to save inexpensive Luna input tokens.
- Do not fill the global default context window simply because it exists.
- Keep Luna workers as leaves; they do not spawn subagents.
- If Luna Max reaches a genuinely hard unresolved decision, send the evidence to an explicitly selected Astra escalation instead of adding another manager model.

## GitHub routing

- Important GitHub work is explicitly delegated to an existing suitable Luna worker before broad exploration, implementation, debugging, or testing; reuse that worker when possible.
- `luna_max` is the default delegated worker for repository search, diff analysis, code or documentation changes, tests, GitHub CLI preparation, and issue/PR drafts; use lower-effort roles only when the task explicitly warrants them, and use `luna_review` for independent review.
- Dispatch independent investigations, test suites, and reviews concurrently to separate Luna workers; keep dependent work sequential and wait for its prerequisites.
- Luna Max owns task framing, final scope, integration, and acceptance; an explicitly selected Astra escalation may handle hard decisions or public repository operations.
- While workers run, Luna Max does not repeat the same scope; independent work units may be delegated to Luna in parallel.
- Never publish secrets, local configuration, credentials, or an unreviewed backlog.

## Compatibility note

The global Codex configuration requests `model_context_window = 1_000_000` and `model_auto_compact_token_limit = 900_000` for the GPT-6 Luna primary session and all five canonical Luna roles. The published model context window is 1.05M tokens, but the Codex runtime or account may expose a smaller effective window; inspect the active session instead of assuming the model maximum. Role files do not override these settings. “Luna Ultra” remains a profile name, not a model or reasoning-effort value.

## License

MIT License. See [`LICENSE`](./LICENSE).

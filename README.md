# Luna_1M_Ultra

A lightweight Codex multi-agent configuration and prompt/routing profile built around GPT-6 Luna as the primary model:

- **Luna Max** — the `gpt-6-luna` primary and default worker, using max reasoning for orchestration, implementation, debugging, testing, and review
- **Astra** — an optional `gpt-6-astra` escalation for explicitly selected work that needs a higher-capability model

`Luna_1M_Ultra` (or “Luna Ultra”) is a conceptual profile name for the prompt, role, context, and routing behavior described here. It is not a model ID or an official model offering. The configured Luna model is `gpt-6-luna`.

This repository is an independent public clone of the profile documentation, not a same-account Git fork.

The profile keeps GPT-6 Luna as the primary session and default delegated model. It uses four predefined Luna roles as leaf workers and allows the main model to invoke them when useful.

## Why this setup

A common multi-agent pattern adds multiple model tiers between orchestration and implementation. That can duplicate context, repeat repository exploration, and add handoff overhead.

This configuration keeps the hierarchy simple:

```text
Luna Max primary (`gpt-6-luna`; max reasoning)
├─ luna_medium (`gpt-6-luna`)  → bounded analysis and lookup
├─ luna_high (`gpt-6-luna`)    → localized fixes / routine checks
├─ luna_max (`gpt-6-luna`)     → implementation / difficult debugging
└─ luna_review (`gpt-6-luna`)  → independent review
```

GPT-6 Luna is designed for efficient, focused, high-volume work. The primary model can dispatch independent tasks to predefined roles and consolidate their results.

## Features

- Luna Max as the single top-level orchestrator
- Luna-only delegated leaf agents
- Astra as an optional, explicitly selected model for work that warrants it
- `Luna_1M_Ultra` as a conceptual prompt/routing profile, not a model ID
- Global default request of `model_context_window = 1_000_000` and `model_auto_compact_token_limit = 900_000`
- Four predefined Luna roles inherit the global context and compaction defaults: `medium`, `high`, `max`, and `review`
- GPT-6 Luna’s official context window is 1.05M; this installed Codex catalog currently caps it at `max_context_window = 872_000`, with an effective runtime compaction limit of `828_400`
- Context reuse across investigation → implementation → testing → revision
- No mandatory reasoning ladder
- No Sol/Terra escalation layer
- Separate read-only review roles

## Setup

See [Luna_1M_Ultra_Setup.md](./Luna_1M_Ultra_Setup.md).

At a minimum, the setup adds:

```toml
model = "gpt-6-luna"
model_reasoning_effort = "max"
plan_mode_reasoning_effort = "max"
review_model = "gpt-6-luna"
model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000

[desktop]
followUpQueueMode = "queue"
show-context-window-usage = true
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

The four role files under `~/.codex/agents/` inherit the global context and compaction settings above; they do not override either setting. Each role uses `gpt-6-luna`; the profile name does not replace the model ID. The setup defines these roles:

```text
~/.codex/agents/
├── luna_medium.toml
├── luna_high.toml
├── luna_max.toml
├── luna_review.toml
```

`expose_spawn_agent_model_overrides = true` makes `spawn_agent` model override controls visible to the orchestrator; it exposes configuration choices, not hidden chain-of-thought.

GPT-6 Luna supports reasoning through `max`, so the primary and `luna_max` roles use `max`. `ultra` remains enabled in the model picker for models that support it, but GPT-6 Luna does not support `ultra`.

The four role names are predefined task classes. The main model may call the suitable role when useful; the screenshot below records six simultaneous work units, not six role definitions.

The TUI footer displays the active model/reasoning, remaining context, and current directory. The `context-remaining` item enables the context-usage UI.

The app’s Settings > General > Context length usage toggle is enabled in this setup. Its config key is `[desktop].show-context-window-usage = true`; the terminal `tui.status_line` is separate. In the current desktop build, the gauge also needs a context-usage percentage for the session and enough composer width. If it is enabled but missing, widen the app window or collapse the sidebar. See the [Codex config example](https://learn.chatgpt.com/docs/config-file/config-sample).

The default follow-up behavior is `queue`, so additional prompts wait until the current run finishes. Use `followUpQueueMode = "queue"`; `stack` is not supported. The app also exposes Queue and Steer under Settings > General. See [desktop settings](https://learn.chatgpt.com/docs/reference/settings).

## Role selection

| Role | Effort | Context | Use case |
|---|---:|---:|---|
| `luna_medium` | medium | Global default | flow tracing, logs, dependencies |
| `luna_high` | high | Global default | small fixes, routine tests |
| `luna_max` | max | Global default | substantive implementation, hard debugging |
| `luna_review` | xhigh | Global default | independent deep review |

These are **task classes, not sequential stages**.

For each task, dispatch independent investigations, test suites, and reviews concurrently to separate Luna workers. Run dependent work sequentially: wait for the prerequisite result before dispatching the next step.
The default delegated role is `luna_max` on `gpt-6-luna` with max reasoning; use a lower-effort predefined role when the task fits it better.

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
- If a Luna role reaches a hard unresolved decision, the primary Luna Max can decide or explicitly select Astra for escalation.

## GitHub routing

- Important GitHub work is explicitly delegated to an existing suitable Luna worker before broad exploration, implementation, debugging, or testing; reuse that worker when possible.
- `luna_max` is the default delegated worker for repository search, diff analysis, code or documentation changes, tests, GitHub CLI preparation, and issue/PR drafts; use lower-effort roles only when the task explicitly warrants them, and use `luna_review` for independent review.
- Dispatch independent investigations, test suites, and reviews concurrently to separate Luna workers; keep dependent work sequential and wait for its prerequisites.
- Luna Max owns task framing, final scope, integration, and acceptance; Astra is an explicit optional escalation.
- While workers run, Luna Max does not repeat the same scope; independent work units may be delegated to Luna in parallel.
- Never publish secrets, local configuration, credentials, or an unreviewed backlog.

## Compatibility note

The global Codex configuration requests `model_context_window = 1_000_000` and `model_auto_compact_token_limit = 900_000` for the GPT-6 Luna primary session and all four predefined roles. GPT-6 Luna’s official context window is 1.05M, while this installed Codex model catalog currently reports `max_context_window = 872_000` and an effective runtime compaction limit of `828_400`. Role files inherit the global context settings.

## License

MIT License. See [`LICENSE`](./LICENSE).

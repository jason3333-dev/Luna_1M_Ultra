# Luna_1M_Ultra — GPT-6 Luna Setup Guide

A Codex multi-agent setup and prompt/routing profile named **Luna_1M_Ultra**, using **GPT-6 Luna (`gpt-6-luna`) with max reasoning as the primary session and default delegated model**.

`Luna_1M_Ultra` (or “Luna Ultra”) is a conceptual profile name, not a model ID or official model offering. The actual model ID for this configuration is `gpt-6-luna`.

This repository is an independent public clone of the profile documentation, not a same-account Git fork.

The design goals are:

- Keep orchestration and integration in Luna Max (`gpt-6-luna` with max reasoning).
- Use `gpt-6-luna` for the primary session and all four predefined Luna roles.
- Let the primary session call the predefined roles when independent work can improve turnaround or coverage.
- Select Luna reasoning effort by task complexity instead of running every task at `max`.
- Reuse relevant Luna context across investigation, implementation, and verification.
- Keep delegated roles as leaves. Astra remains an optional, explicitly selected escalation.
- Request the global context and compaction defaults for Luna Max and all four Luna roles; the current model catalog may clamp those requests.

> GPT-6 Luna’s official context window is 1.05M. This configuration requests `model_context_window = 1_000_000` and `model_auto_compact_token_limit = 900_000`. The installed Codex model catalog currently caps the request at `max_context_window = 872_000`, with an effective runtime compaction limit of `828_400`.

> GPT-6 Luna supports reasoning through `max`, but not `ultra`. Keep `ultra` enabled in the picker for other models that support it; use `max` for the GPT-6 Luna primary and roles.

---

## Architecture

```text
Luna Max primary (`gpt-6-luna`; max reasoning)
├─ luna_medium (`gpt-6-luna`)  → bounded lookup, flow and dependency analysis
├─ luna_high (`gpt-6-luna`)    → small fixes, routine checks
├─ luna_max (`gpt-6-luna`)     → substantive implementation, difficult debugging
└─ luna_review (`gpt-6-luna`)  → independent review
```

The roles are independent task classes, not a mandatory sequence. Pick the role that matches the task.
The default delegated role is `luna_max` with max reasoning; use a lower-effort role only when the task explicitly warrants it.

For each task, dispatch independent investigations, test suites, and reviews concurrently to separate Luna workers. Run dependent work sequentially, waiting for each prerequisite result before dispatching the next step.

For difficult questions, the primary Luna Max decides directly by default. Astra can be selected explicitly for an escalation. Predefined Luna roles are leaves and do not spawn subagents.

---

## 1. Global `config.toml` for the `Luna_1M_Ultra` profile

Merge the following into `~/.codex/config.toml` or the active Codex configuration.

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

Notes:

- `max_depth` applies to older V1 multi-agent behavior. V2 may ignore it.
- `expose_spawn_agent_model_overrides = true` makes `spawn_agent` model override controls visible to the orchestrator; it exposes configuration choices, not hidden chain-of-thought.
- The global context and compaction request applies to the GPT-6 Luna primary and all four canonical Luna roles.
- GPT-6 Luna’s official context window is 1.05M. The installed local model catalog currently caps it at `max_context_window = 872_000`, with an effective runtime compaction limit of `828_400`.
- The profile name is not a replacement for a model ID: use `gpt-6-luna` with max reasoning.
- The desktop reasoning-effort availability list includes `max` and `ultra`: `enabled-reasoning-efforts = ["low", "medium", "high", "xhigh", "persistent", "max", "ultra"]`.
- Both `max` and `ultra` remain enabled in the model picker. GPT-6 Luna supports `max` but not `ultra`, so use `max` for this profile.
- Reasoning effort is a setting separate from the model ID; use `gpt-6-luna` in model fields.
- The TUI footer uses `status_line = ["model-with-reasoning", "context-remaining", "current-dir"]`; `context-remaining` enables the context-usage display.
- `[desktop].show-context-window-usage = true` enables the context-window usage indicator in the app composer; this is separate from the terminal TUI footer setting.
- `followUpQueueMode = "queue"` holds a follow-up prompt until the current run finishes; `steer` applies it to the active run. Use the exact value `queue`—`stack` is not supported.

### App context gauge troubleshooting

- `[desktop].show-context-window-usage = true` enables the composer context gauge. `tui.status_line` configures only the terminal TUI footer.
- The current desktop build renders the gauge when the session has a context-usage percentage and the responsive composer layout has room for it. A narrow window or expanded sidebar may hide it; widen the window or collapse the sidebar.
- Settings > General > Context length usage is enabled in this setup. If the composer has room but the gauge is still absent, the active thread may not be supplying usage metadata.

---

## 2. Luna roles in the `Luna_1M_Ultra` routing profile

Create these files under `~/.codex/agents/`.

These four role files intentionally omit `model_context_window` and `model_auto_compact_token_limit`; every canonical role inherits the global defaults from section 1. The local model catalog may clamp those inherited requests as described above. Each file uses `gpt-6-luna`; `Luna_1M_Ultra` is only the conceptual profile name.

### `luna_medium.toml`

```toml
name = "luna_medium"
description = "Luna medium: bounded call-flow, log, dependency and root-cause analysis. Read-only."
model = "gpt-6-luna"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
sandbox_mode = "read-only"

developer_instructions = """
Trace the smallest relevant execution path.
Reuse supplied evidence instead of repeating broad scans.
Separate confirmed behavior from hypotheses and identify decisive checks.
Return a bounded diagnosis or implementation plan.
Do not edit files or spawn subagents.
"""
```

### `luna_high.toml`

```toml
name = "luna_high"
description = "Luna high: clear localized fixes, routine tests, and small refactors."
model = "gpt-6-luna"
model_reasoning_effort = "high"
plan_mode_reasoning_effort = "high"
sandbox_mode = "workspace-write"

developer_instructions = """
Implement the bounded change and run focused checks.
Add regression coverage where appropriate.
Continue investigation, testing, and revisions in this thread instead of splitting every step into a new worker.
Report architectural ambiguity or high-risk changes to the primary Luna Max session; use Astra only when explicitly selected.
Do not spawn subagents.
"""
```

### `luna_max.toml`

```toml
name = "luna_max"
description = "Luna max: substantive implementation, difficult debugging, and multi-file changes."
model = "gpt-6-luna"
model_reasoning_effort = "max"
plan_mode_reasoning_effort = "max"
sandbox_mode = "workspace-write"

developer_instructions = """
Own a coherent engineering task from investigation through implementation, tests, and revisions.
Reuse retained context and completed investigation.
Do ordinary searches needed by the implementation yourself.
If two materially different approaches fail without meaningful progress, report the exact unresolved question and evidence to the primary Luna Max session.
After the primary Luna Max session decides, continue implementation and verification in this same thread.
Do not spawn subagents.
"""
```

### `luna_review.toml`

```toml
name = "luna_review"
description = "Luna review: independent review of diffs, behavior, regressions, and test coverage. Read-only."
model = "gpt-6-luna"
model_reasoning_effort = "xhigh"
plan_mode_reasoning_effort = "xhigh"
sandbox_mode = "read-only"

developer_instructions = """
Review the actual diff and affected implementation independently.
Prioritize correctness, regressions, and missing assertions over style.
Support findings with file references and reproduction or test ideas.
Do not edit source or tests. Do not spawn subagents.
"""
```

---

## 3. Canonical `Luna_1M_Ultra` role set

The active `~/.codex/agents/` directory contains exactly these four canonical Luna roles:

| Role | Effort | Context | Permissions |
|---|---:|---:|---|
| `luna_medium` | medium | Global default | read-only |
| `luna_high` | high | Global default | workspace-write |
| `luna_max` | max | Global default | workspace-write |
| `luna_review` | xhigh | Global default | read-only |

Do not add compatibility aliases to the active role directory unless a specific caller still requires one.

---

## 4. `AGENTS.md` routing block for `Luna_1M_Ultra`

Add this block to the active global or project `AGENTS.md`.

```markdown
<!-- BEGIN ASTRA_LUNA_1M -->
## Luna Max routing

Luna Max is the primary/default session and owns orchestration, hard technical decisions, integration, and final acceptance.
All delegated AI work uses Luna. Astra is an optional explicit escalation, not the default primary model. Do not use Sol, Terra, or Astra child sessions.

For important repository work, delegate one coherent unit to an existing suitable Luna worker before broad exploration, implementation, debugging, or testing. Reuse a useful worker before creating a new one. Dispatch independent investigations, test suites, and reviews concurrently to separate Luna workers. Run dependent work sequentially, waiting for prerequisite results before dispatching the next step. While workers run, Luna Max does not repeat the same scope.

The default delegated worker is `luna_max` with max reasoning. Use lower-effort roles only when the task explicitly warrants them.

Use a small number of parallel Luna workers for independent units that improve turnaround or coverage. Do not create duplicate or ceremonial workers, and do not let workers edit the same file concurrently.

Choose the Luna role directly by task difficulty and permission needs:
- `luna_medium`: medium — bounded flow/log/dependency analysis.
- `luna_high`: high — clear small fixes and routine tests.
- `luna_max`: max — substantive implementation and difficult debugging.
- `luna_review`: xhigh — independent deep review with read-only permissions.

Do not run roles as a mandatory reasoning ladder.
A Luna max worker performs ordinary searches required by its implementation.
Luna workers are leaves and do not spawn subagents.

Prefer reusing the same Luna worker for related investigation, implementation, testing, and revisions.
Do not close a useful worker between phases of the same coherent task.
For unrelated or one-off work, start a fresh worker with only the needed task context.
Do not fill the global default context window simply because it is available.

When the spawn API exposes history controls, choose them intentionally:
- V2: `fork_turns = "none" | "all" | "<positive integer>"`
- V1: `fork_context = false | true`
Never send both forms in the same spawn call.

After two materially different failed approaches without progress, Luna reports the exact unresolved question to Luna Max.
Luna Max decides directly by default, then the same Luna worker continues implementation and verification. An explicitly selected Astra escalation may decide only when requested.
Escalate security or data-loss risks immediately.

Luna Max may act directly for simple answers, tiny obvious work, judgments requiring the full primary-session context, or a concrete technical or permission blocker to delegation. Do not skip delegation merely because the primary session can do the work.

Use a small useful number of independent workers.
Do not let multiple workers edit the same file concurrently.
Use independent review when it materially improves correctness; do not create ceremonial reviewers.
Do not claim unrun tests passed.

GitHub routing:
- Route important GitHub work to the default `luna_max` worker: use it for repository search, diff analysis, code or documentation changes, tests, GitHub CLI preparation, and issue/PR drafting; use `luna_review` for independent review.
- Keep optional Astra escalations to the minimum needed for explicitly requested hard decisions or public repository operations; Luna Max remains the default owner.
- Never publish secrets, local configuration, credentials, or an unreviewed backlog.
<!-- END ASTRA_LUNA_1M -->
```

---

## 5. `Luna_1M_Ultra` context strategy

The setup intentionally does **not** force every Luna task to start from empty context.

Recommended behavior:

| Situation | Context strategy |
|---|---|
| One-off file or symbol lookup | Handle directly in the primary session; use `luna_medium` for a bounded investigation |
| Dependent investigation → implementation → revisions | Reuse the same Luna worker sequentially |
| Independent investigation, test suite, or review | Dispatch concurrently to separate Luna workers |
| Unrelated task | Start fresh |
| Hard unresolved decision | Report the issue and evidence to the primary Luna Max session; use Astra only when explicitly selected |

The goal is to minimize duplicated reasoning and re-exploration, not merely to minimize raw Luna input tokens.

---

## 6. `Luna_1M_Ultra` verification

After applying the configuration, verify:

1. Primary session uses `gpt-6-luna` with max reasoning and requests the global `model_context_window = 1_000_000` and `model_auto_compact_token_limit = 900_000` defaults.
2. Each canonical Luna role inherits those global context and compaction settings without role-specific overrides.
3. `luna_max` resolves to `gpt-6-luna`/max.
4. `luna_review` resolves to `gpt-6-luna`/xhigh.
5. The official GPT-6 Luna context window is 1.05M; this installed local catalog reports `max_context_window = 872_000` and an effective runtime compaction limit of `828_400`.
6. Existing project/profile overrides do not silently replace the model or reasoning settings.
7. Sol/Terra and Astra child sessions are not selected by default in the routing configuration.
8. `Luna_1M_Ultra` remains a conceptual prompt/routing alias and is not used as a `model` value.
9. The desktop reasoning-effort availability list includes `max` and `ultra`.
10. GPT-6 Luna uses `max` and `xhigh`; do not select `ultra` because this model does not support it.
11. The TUI footer includes the `context-remaining` item.
12. Desktop follow-up prompt behavior is set to `queue`.
13. The app composer context-window usage indicator is enabled.

Do not use the model's self-reported identity as the only verification source; prefer actual session/model metadata when available.

---

## 7. `Luna_1M_Ultra` notes

- The global configuration requests `model_context_window = 1_000_000` and `model_auto_compact_token_limit = 900_000` for GPT-6 Luna; all four predefined roles inherit those settings.
- GPT-6 Luna’s official context window is 1.05M; the installed local model catalog currently clamps it to `max_context_window = 872_000`, with an effective runtime compaction limit of `828_400`.
- A requested context value does not guarantee that every Codex build/account exposes the full capacity.
- Prompt/cache reuse is conditional and should not be assumed from thread reuse alone.
- `AGENTS.md` is routing guidance, not a hard model allow-list or security boundary.
- Keep permissions, provider settings, MCP configuration, and unrelated project configuration unchanged unless you intentionally manage them separately.

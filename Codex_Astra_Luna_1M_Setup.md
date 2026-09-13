# Codex Astra + Luna Model-Max Context Setup

A minimal Codex multi-agent setup using **Astra as the primary orchestrator** and **Luna as the only delegated worker model**.

The design goals are:

- Keep orchestration and hard technical decisions in Astra.
- Use Luna for exploration, implementation, debugging, testing, and review.
- Because Luna is low-cost, use parallel Luna workers proactively when independent work units can improve turnaround or coverage.
- Select Luna reasoning effort by task complexity instead of running every task at `max`.
- Reuse relevant Luna context across investigation, implementation, and verification.
- Avoid unnecessary Sol/Terra handoffs and duplicated high-cost context.
- Use Astra's native default context and request the maximum context allowed by the Codex model catalog for each Luna role.

> Astra leaves context and compaction at Codex defaults. Each Luna role requests the maximum context allowed by the current Codex model catalog; effective limits are still bounded by the installed Codex build and account availability.

---

## Architecture

```text
Astra main/native (~258K default)
├─ Luna Low      Codex max → file/symbol lookup, narrow searches
├─ Luna Medium   Codex max → flow tracing, logs, dependency analysis
├─ Luna High     Codex max → small fixes, routine tests
├─ Luna Max      Codex max → substantive implementation, difficult debugging
└─ Luna Review   Codex max → independent review
```

There is no mandatory `low → medium → high → max` pipeline. Pick the role that matches the task.

For difficult questions, Luna reports the unresolved issue back to the existing Astra session. Astra decides directly, then the same Luna worker can continue implementation and verification.

---

## 1. Global `config.toml`

Merge the following into `~/.codex/config.toml` or the active Codex configuration.

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
max_depth = 1
```

Notes:

- `max_depth` applies to older V1 multi-agent behavior. V2 may ignore it.
- Astra context and compaction remain at Codex defaults.
- Luna role files request the current Codex model maximum with its corresponding compaction limit; the catalog may lower that maximum.

---

## 2. Luna roles

Create these files under `~/.codex/agents/`.

### `luna_low.toml`

```toml
name = "luna_low"
description = "Luna low: exact file/symbol lookup, references, and small factual searches. Read-only."
model = "gpt-5.6-luna"
model_reasoning_effort = "low"
plan_mode_reasoning_effort = "low"
model_context_window = 872_000
model_auto_compact_token_limit = 828_400
sandbox_mode = "read-only"

developer_instructions = """
Find the requested facts with scoped searches and relevant file ranges.
Batch related lookups. Do not map the whole repository unless needed.
Do not modify files or perform speculative redesign.
Return concise findings, relevant paths, exact checks, and blockers.
Do not spawn subagents or call other AI models.
"""
```

### `luna_medium.toml`

```toml
name = "luna_medium"
description = "Luna medium: bounded call-flow, log, dependency and root-cause analysis. Read-only."
model = "gpt-5.6-luna"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_context_window = 872_000
model_auto_compact_token_limit = 828_400
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
model = "gpt-5.6-luna"
model_reasoning_effort = "high"
plan_mode_reasoning_effort = "high"
model_context_window = 872_000
model_auto_compact_token_limit = 828_400
sandbox_mode = "workspace-write"

developer_instructions = """
Implement the bounded change and run focused checks.
Add regression coverage where appropriate.
Continue investigation, testing, and revisions in this thread instead of splitting every step into a new worker.
Report architectural ambiguity or high-risk changes to Astra.
Do not spawn subagents.
"""
```

### `luna_max.toml`

```toml
name = "luna_max"
description = "Luna max: substantive implementation, difficult debugging, and multi-file changes."
model = "gpt-5.6-luna"
model_reasoning_effort = "max"
plan_mode_reasoning_effort = "max"
model_context_window = 872_000
model_auto_compact_token_limit = 828_400
sandbox_mode = "workspace-write"

developer_instructions = """
Own a coherent engineering task from investigation through implementation, tests, and revisions.
Reuse retained context and completed investigation.
Do ordinary searches needed by the implementation yourself.
If two materially different approaches fail without meaningful progress, report the exact unresolved question and evidence to Astra.
After Astra decides, continue implementation and verification in this same thread.
Do not spawn subagents.
"""
```

### `luna_review.toml`

```toml
name = "luna_review"
description = "Luna review: independent review of diffs, behavior, regressions, and test coverage. Read-only."
model = "gpt-5.6-luna"
model_reasoning_effort = "high"
plan_mode_reasoning_effort = "high"
model_context_window = 872_000
model_auto_compact_token_limit = 828_400
sandbox_mode = "read-only"

developer_instructions = """
Review the actual diff and affected implementation independently.
Prioritize correctness, regressions, and missing assertions over style.
Support findings with file references and reproduction or test ideas.
Do not edit source or tests. Do not spawn subagents.
"""
```

---

## 3. Canonical role set

The active `~/.codex/agents/` directory contains exactly these five canonical Luna roles:

| Role | Effort | Context | Permissions |
|---|---:|---:|---|
| `luna_low` | low | Codex max | read-only |
| `luna_medium` | medium | Codex max | read-only |
| `luna_high` | high | Codex max | workspace-write |
| `luna_max` | max | Codex max | workspace-write |
| `luna_review` | high | Codex max | read-only |

Do not add compatibility aliases to the active role directory unless a specific caller still requires one.

---

## 4. `AGENTS.md` routing block

Add this block to the active global or project `AGENTS.md`.

```markdown
<!-- BEGIN ASTRA_LUNA_1M -->
## Astra + Luna routing

Astra is the main session and owns orchestration, hard technical decisions, integration, and final acceptance.
All delegated AI work uses Luna. Do not use Sol, Terra, or another Astra child.

For important repository work, delegate one coherent unit to an existing suitable Luna worker before broad exploration, implementation, debugging, or testing. Reuse a useful worker before creating a new one. Delegate even when only one task can be parallelized; delegate independent units in parallel. While Luna works, Astra does not repeat the same scope.

Because Luna is low-cost, actively use a small number of parallel Luna workers when independent units can improve turnaround or coverage. Do not create duplicate or ceremonial workers, and do not let workers edit the same file concurrently.

Choose the Luna role directly by task difficulty and permission needs:
- `luna_low`: low — exact searches and facts.
- `luna_medium`: medium — bounded flow/log/dependency analysis.
- `luna_high`: high — clear small fixes and routine tests.
- `luna_max`: max — substantive implementation and difficult debugging.
- `luna_review`: high — independent review with read-only permissions.

Do not run low → medium → high → max as mandatory stages.
A Luna max worker performs ordinary searches required by its implementation.
Luna workers are leaves and do not spawn subagents.

Prefer reusing the same Luna worker for related investigation, implementation, testing, and revisions.
Do not close a useful worker between phases of the same coherent task.
For unrelated or one-off work, start a fresh worker with only the needed task context.
Do not fill the native-default Astra window or a Luna model-maximum window just because it is available.

When the spawn API exposes history controls, choose them intentionally:
- V2: `fork_turns = "none" | "all" | "<positive integer>"`
- V1: `fork_context = false | true`
Never send both forms in the same spawn call.

After two materially different failed approaches without progress, Luna reports the exact unresolved question to Astra.
Astra decides directly, then the same Luna worker continues implementation and verification.
Escalate security or data-loss risks immediately.

Astra may act directly only for simple answers, tiny obvious work, judgments requiring the full main-session context, or a concrete technical or permission blocker to delegation. Do not skip delegation merely because Astra can do the work.

Use a small useful number of independent workers.
Do not let multiple workers edit the same file concurrently.
Use independent review when it materially improves correctness; do not create ceremonial reviewers.
Do not claim unrun tests passed.

GitHub routing:
- Route important GitHub work to Luna/max by default: use `luna_max` for repository search, diff analysis, code or documentation changes, tests, GitHub CLI preparation, and issue/PR drafting; use `luna_review` for independent review.
- Keep Astra to the minimum needed for task framing, final scope and safety approval, and execution of public repository creation, pushes, merges, and permission changes.
- Never publish secrets, local configuration, credentials, or an unreviewed backlog.
<!-- END ASTRA_LUNA_1M -->
```

---

## 5. Context strategy

The setup intentionally does **not** force every Luna task to start from empty context.

Recommended behavior:

| Situation | Context strategy |
|---|---|
| One-off file or symbol lookup | Fresh `luna_low` with minimal task context |
| Investigation → implementation → tests → revisions | Reuse the same Luna worker |
| Related parallel task | Inherit only useful history when supported |
| Unrelated task | Start fresh |
| Hard unresolved decision | Report the issue and evidence to Astra; do not add another manager model |

The goal is to minimize duplicated reasoning and re-exploration, not merely to minimize raw Luna input tokens.

---

## 6. Verification

After applying the configuration, verify:

1. Primary session runs Astra with the Codex native-default context (approximately 258K in this setup).
2. Each canonical Luna role requests the maximum context allowed by the current Codex model catalog.
3. `luna_low` resolves to Luna/low.
4. `luna_max` resolves to Luna/max.
5. The effective runtime/model-catalog maximum is not lower than expected.
6. Existing project/profile overrides do not silently replace the model or reasoning settings.
7. Sol/Terra are not selected by the routing configuration.

Do not use the model's self-reported identity as the only verification source; prefer actual session/model metadata when available.

---

## 7. Notes

- Luna role files request the maximum context allowed by the current Codex model catalog; Astra leaves context and compaction at Codex defaults.
- A requested context value does not guarantee that every Codex build/account exposes the full capacity.
- Prompt/cache reuse is conditional and should not be assumed from thread reuse alone.
- `AGENTS.md` is routing guidance, not a hard model allow-list or security boundary.
- Keep permissions, provider settings, MCP configuration, and unrelated project configuration unchanged unless you intentionally manage them separately.

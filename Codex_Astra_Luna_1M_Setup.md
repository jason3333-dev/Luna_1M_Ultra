# Codex Astra + Luna 1M Context Setup

A minimal Codex multi-agent setup using **Astra as the primary orchestrator** and **Luna as the only delegated worker model**.

The design goals are:

- Keep orchestration and hard technical decisions in Astra.
- Use Luna for exploration, implementation, debugging, testing, and review.
- Select Luna reasoning effort by task complexity instead of running every task at `max`.
- Reuse relevant Luna context across investigation, implementation, and verification.
- Avoid unnecessary Sol/Terra handoffs and duplicated high-cost context.
- Request a 1M context window for the primary session and Luna roles.

> `model_context_window = 1_000_000` is a requested capacity. The effective runtime limit is still bounded by the model catalog and the installed Codex build.

---

## Architecture

```text
Astra max
├─ Luna low       → file/symbol lookup, narrow searches
├─ Luna medium    → flow tracing, logs, dependency analysis
├─ Luna high      → small fixes, routine tests, standard review
└─ Luna max       → substantive implementation, difficult debugging,
                    multi-file work, deep review
```

There is no mandatory `low → medium → high → max` pipeline. Pick the role that matches the task.

For difficult questions, Luna reports the unresolved issue back to the existing Astra session. Astra decides directly, then the same Luna worker can continue implementation and verification.

---

## 1. Global `config.toml`

Merge the following into `~/.codex/config.toml` or the active Codex configuration.

```toml
model = "gpt-6-astra"
model_reasoning_effort = "max"
plan_mode_reasoning_effort = "max"
review_model = "gpt-5.6-luna"

model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000

[agents]
enabled = true
default_subagent_model = "gpt-5.6-luna"
default_subagent_reasoning_effort = "medium"
max_concurrent_threads_per_session = 12
max_depth = 1
```

Notes:

- `max_depth` applies to older V1 multi-agent behavior. V2 may ignore it.
- The 900K compaction threshold is an operating starting point, not a universal optimum.
- If the model catalog exposes a lower maximum context window, Codex may clamp the requested value.

---

## 2. Luna roles

Create these files under `~/.codex/agents/`.

### `luna_scan.toml`

```toml
name = "luna_scan"
description = "Luna low: exact file/symbol lookup, references, and small factual searches. Read-only."
model = "gpt-5.6-luna"
model_reasoning_effort = "low"
plan_mode_reasoning_effort = "low"
model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000
sandbox_mode = "read-only"

developer_instructions = """
Find the requested facts with scoped searches and relevant file ranges.
Batch related lookups. Do not map the whole repository unless needed.
Do not modify files or perform speculative redesign.
Return concise findings, relevant paths, exact checks, and blockers.
Do not spawn subagents or call other AI models.
"""
```

### `luna_analyze.toml`

```toml
name = "luna_analyze"
description = "Luna medium: bounded call-flow, log, dependency and root-cause analysis. Read-only."
model = "gpt-5.6-luna"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000
sandbox_mode = "read-only"

developer_instructions = """
Trace the smallest relevant execution path.
Reuse supplied evidence instead of repeating broad scans.
Separate confirmed behavior from hypotheses and identify decisive checks.
Return a bounded diagnosis or implementation plan.
Do not edit files or spawn subagents.
"""
```

### `luna_quickfix.toml`

```toml
name = "luna_quickfix"
description = "Luna high: clear localized fixes, routine tests, and small refactors."
model = "gpt-5.6-luna"
model_reasoning_effort = "high"
plan_mode_reasoning_effort = "high"
model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000
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
model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000
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
description = "Luna high: independent review of diffs, behavior, regressions, and test coverage."
model = "gpt-5.6-luna"
model_reasoning_effort = "high"
plan_mode_reasoning_effort = "high"
model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000
sandbox_mode = "read-only"

developer_instructions = """
Review the actual diff and affected implementation independently.
Prioritize correctness, regressions, and missing assertions over style.
Support findings with file references and reproduction or test ideas.
Do not edit source or tests. Do not spawn subagents.
"""
```

### `luna_review_max.toml`

```toml
name = "luna_review_max"
description = "Luna max: deep review of concurrency, state, recovery, and invariant risks."
model = "gpt-5.6-luna"
model_reasoning_effort = "max"
plan_mode_reasoning_effort = "max"
model_context_window = 1_000_000
model_auto_compact_token_limit = 900_000
sandbox_mode = "read-only"

developer_instructions = """
Investigate the specific difficult correctness risk independently.
Examine invariants, interleavings, recovery paths, and counterexamples.
Read the actual code and relevant tests rather than relying on summaries.
Do not edit files or spawn subagents.
Return supported defects, untested assumptions, and the smallest additional verification needed.
"""
```

---

## 3. Built-in role aliases

To keep built-in role names on Luna, create equivalent aliases:

| Role | Base | Effort |
|---|---|---|
| `default` | `luna_analyze` | medium |
| `worker` | `luna_max` | max |
| `explorer` | `luna_scan` | low |

Copy the corresponding role file, change only `name`, and optionally prefix the description with `Alias of ...`.

If your setup already defines `reviewer`, map it to the same model/settings as `luna_review`.

---

## 4. `AGENTS.md` routing block

Add this block to the active global or project `AGENTS.md`.

```markdown
<!-- BEGIN ASTRA_LUNA_1M -->
## Astra + Luna routing

Astra owns orchestration, hard technical decisions, integration, and final acceptance.
All delegated AI work uses Luna. Do not use Sol, Terra, or another Astra child.

Choose the Luna role directly by task size:
- `luna_scan`: low — exact searches and facts.
- `luna_analyze`: medium — bounded flow/log/dependency analysis.
- `luna_quickfix`: high — clear small fixes and routine tests.
- `luna_max`: max — substantive implementation and difficult debugging.
- `luna_review`: high — normal independent review.
- `luna_review_max`: max — targeted deep correctness review.

Do not run low → medium → high → max as mandatory stages.
A Luna max worker performs ordinary searches required by its implementation.
Luna workers are leaves and do not spawn subagents.

Prefer reusing the same Luna worker for related investigation, implementation, testing, and revisions.
Do not close a useful worker between phases of the same coherent task.
For unrelated or one-off work, start a fresh worker with only the needed task context.
Do not fill the 1M window just because it is available.

When the spawn API exposes history controls, choose them intentionally:
- V2: `fork_turns = "none" | "all" | "<positive integer>"`
- V1: `fork_context = false | true`
Never send both forms in the same spawn call.

After two materially different failed approaches without progress, Luna reports the exact unresolved question to Astra.
Astra decides directly, then the same Luna worker continues implementation and verification.
Escalate security or data-loss risks immediately.

Use a small useful number of independent workers.
Do not let multiple workers edit the same file concurrently.
Use independent review when it materially improves correctness; do not create ceremonial reviewers.
Do not claim unrun tests passed.
<!-- END ASTRA_LUNA_1M -->
```

---

## 5. Context strategy

The setup intentionally does **not** force every Luna task to start from empty context.

Recommended behavior:

| Situation | Context strategy |
|---|---|
| One-off file or symbol lookup | Fresh `luna_scan` with minimal task context |
| Investigation → implementation → tests → revisions | Reuse the same Luna worker |
| Related parallel task | Inherit only useful history when supported |
| Unrelated task | Start fresh |
| Hard unresolved decision | Report the issue and evidence to Astra; do not add another manager model |

The goal is to minimize duplicated reasoning and re-exploration, not merely to minimize raw Luna input tokens.

---

## 6. Verification

After applying the configuration, verify:

1. Primary session runs Astra at the requested reasoning effort.
2. `luna_scan` resolves to Luna/low.
3. `luna_max` resolves to Luna/max.
4. Requested context is `1_000_000` for the primary session and Luna roles.
5. The effective runtime/model-catalog maximum is not lower than expected.
6. Existing project/profile overrides do not silently replace the model or reasoning settings.
7. Sol/Terra are not selected by the routing configuration.

Do not use the model's self-reported identity as the only verification source; prefer actual session/model metadata when available.

---

## 7. Notes

- A configured 1M context window does not guarantee that every Codex build/account exposes the full requested capacity.
- Prompt/cache reuse is conditional and should not be assumed from thread reuse alone.
- `AGENTS.md` is routing guidance, not a hard model allow-list or security boundary.
- Keep permissions, provider settings, MCP configuration, and unrelated project configuration unchanged unless you intentionally manage them separately.

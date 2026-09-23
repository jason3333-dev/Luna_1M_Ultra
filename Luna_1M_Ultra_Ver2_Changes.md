# Luna_1M_Ultra v2 — GPT-6 Luna migration

[English](./Luna_1M_Ultra_Ver2_Changes.md) | [한국어](./Luna_1M_Ultra_Ver2_Changes.kr.md)

Date: 2026-09-23

## Changes

- Migrated the primary session and default delegated worker from `gpt-5.6-luna` to `gpt-6-luna`, both with `max` reasoning.
- Removed `luna_low` and kept four Luna roles: `luna_medium`/medium, `luna_high`/high, `luna_max`/max, and `luna_review`/xhigh.
- Requested `model_context_window = 1_000_000` and `model_auto_compact_token_limit = 900_000`. GPT-6 Luna's published context window was 1.05M; at migration, the installed Codex catalog reported `max_context_window = 872_000` and an effective runtime compaction limit of `828_400`.
- Enabled the app's context usage indicator with `[desktop].show-context-window-usage = true` and the terminal TUI indicator with `context-remaining`.
- Corrected `followUpQueueMode` from unsupported `stack` to `queue`, which held new prompts until the active run finished.
- Kept `max` and `ultra` in the reasoning picker. `gpt-6-luna` used `max` and did not support `ultra`.

## README snapshot and usage observation

`README.md` documented a screenshot, `parallel-luna-max-run.png`, of six parallel work units:

1. Core gate — planning-focused test
2. Operator review — halt/resume race analysis
3. Fill execution p1 — delivery identity logic
4. Accounting p1 — filename parsing analysis
5. Pair machine p1 — external increase impact
6. Candidate p1 — parent communication options

The supplied Pro 5X observation was approximately 1% of the weekly usage window per 5 minutes during this kind of burst. It was a planning reference, not an official quota, billing promise, or guaranteed rate.

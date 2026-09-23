# Luna_1M_Ultra v3 — Changes (2026-09-23)

[English](./Luna_1M_Ultra_Ver3_Changes.md) | [한국어](./Luna_1M_Ultra_Ver3_Changes.kr.md)

## Delegation policy

- Broadened proactive delegation from important repository work to every nontrivial task. Enabled concurrent independent research, implementation, tests, and reviews once their inputs are clear, while keeping dependent work sequential.
- Added a distinct analysis or validation workstream for tasks with one main path when it adds value. Limited direct handling to trivial one-step tasks, inseparable or primary-context-only work, cases with no delegation benefit, and safety, privacy, or conflicting-write constraints.
- Updated routing to reuse suitable Luna workers across related phases and keep them as leaves under Luna Max's orchestration. Required distinct scopes and prohibited concurrent edits to the same file.
- Retained the 12-thread concurrency cap as a ceiling. Clarified that six was not a fixed target and prohibited adding workers to fill slots.

## Documentation language policy

- Introduced a complete Korean companion for every repository document. Required `.kr.md` for Markdown copies, `.kr` for extensionless documents, reciprocal language links near the top, and synchronized facts, examples, identifiers, paths, and links.
- Added `LICENSE.kr` as an unofficial Korean translation and kept the English `LICENSE` authoritative.

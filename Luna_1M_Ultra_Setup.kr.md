# Luna_1M_Ultra — GPT-6 Luna 설정 가이드

[English](Luna_1M_Ultra_Setup.md) | [한국어](Luna_1M_Ultra_Setup.kr.md)

**GPT-6 Luna (`gpt-6-luna`)를 최대 추론 강도의 기본 위임 모델이자 기본 세션으로 사용하는** Codex 멀티 에이전트 설정 및 프롬프트/라우팅 프로필인 **Luna_1M_Ultra**입니다.

`Luna_1M_Ultra`(또는 “Luna Ultra”)는 개념적인 프로필 이름이며, 모델 ID나 공식 모델 상품명이 아닙니다. 이 설정의 실제 모델 ID는 `gpt-6-luna`입니다.

이 저장소는 프로필 문서를 독립적으로 공개 복제한 것이며, 동일 계정의 Git 포크가 아닙니다.

## 언어별 문서

모든 문서에 완전한 한국어 대응본을 유지합니다. Markdown 대응본에는 `.kr.md` 접미사를 사용하고 영문 원문으로 연결되는 링크를 둡니다. 두 문서의 사실, 예시, 코드, 설정 키, 경로, 링크를 일치시키고 기술 식별자는 양쪽에서 그대로 유지합니다. `LICENSE.kr`는 편의를 위한 비공식 번역본이며, 충돌이 있을 경우 영문 `LICENSE`가 우선합니다.

설계 목표는 다음과 같습니다.

- 오케스트레이션과 통합은 최대 추론 강도의 Luna Max(`gpt-6-luna`)에서 수행합니다.
- 기본 세션과 미리 정의된 네 가지 Luna 역할 모두에 `gpt-6-luna`를 사용합니다.
- 사소하지 않은 모든 작업에서 조사, 구현, 테스트, 검토에 유용한 독립 작업 흐름을 선제적으로 위임합니다. 작업 흐름이 하나뿐인 경우에는 가치가 있을 때 독립 분석이나 검증을 추가합니다.
- 모든 작업을 `max`로 실행하는 대신 작업 난이도에 따라 Luna의 추론 강도를 선택합니다.
- 조사, 구현, 검증 전반에서 관련 Luna 컨텍스트를 재사용합니다.
- 위임된 역할은 리프 역할로 유지합니다. Astra는 선택적으로 명시해 사용하는 에스컬레이션 경로입니다.
- Luna Max와 네 가지 Luna 역할 모두에 전역 컨텍스트 및 압축 기본값을 요청합니다. 현재 모델 카탈로그가 요청값을 제한할 수 있습니다.

> GPT-6 Luna의 공식 컨텍스트 윈도우는 1.05M입니다. 이 설정은 `model_context_window = 1_000_000` 및 `model_auto_compact_token_limit = 900_000`을 요청합니다. 설치된 Codex 모델 카탈로그는 현재 요청을 `max_context_window = 872_000`으로 제한하며, 런타임에서 적용되는 압축 한도는 `828_400`입니다.

> GPT-6 Luna는 `max`까지 추론을 지원하지만 `ultra`는 지원하지 않습니다. 다른 지원 모델을 위해 선택기에서 `ultra`를 활성화된 상태로 두고, GPT-6 Luna 기본 세션과 역할에는 `max`를 사용합니다.

---

## 아키텍처

```text
Luna Max primary (`gpt-6-luna`; max reasoning)
├─ luna_medium (`gpt-6-luna`)  → bounded lookup, flow and dependency analysis
├─ luna_high (`gpt-6-luna`)    → small fixes, routine checks
├─ luna_max (`gpt-6-luna`)     → substantive implementation, difficult debugging
└─ luna_review (`gpt-6-luna`)  → independent review
```

각 역할은 필수 순서가 아니라 서로 독립적인 작업 유형입니다. 작업에 맞는 역할을 선택하세요.
기본 위임 역할은 최대 추론 강도의 `luna_max`입니다. 범위, 작업량 또는 권한이 더 적합하다면 `luna_medium`, `luna_high` 또는 `luna_review`를 바로 선택하세요. 역할을 순서대로 거치게 하지 마세요.

사소하지 않은 모든 작업에서 입력이 명확해지는 대로 독립 조사, 구현 단위, 테스트, 검토를 병렬로 시작합니다. 의존 작업은 순차적으로 진행합니다. 설정된 최대 12개 스레드 한도 내에서 작업이 뒷받침하는 만큼의 유용하고 독립적인 작업 흐름을 사용합니다. 이 한도는 목표 개수가 아니며, 설정만으로 에이전트가 시작되지는 않습니다. 여섯 개를 목표로 삼거나 형식적인 작업으로 수를 채우지 말고, 중복 범위와 겹치는 편집을 피하세요. 작업 흐름이 하나뿐인 사소하지 않은 작업이라도 가치가 있다면 유용한 독립 분석이나 검증을 위임합니다. 사소한 단일 단계 작업, 본질적으로 분리할 수 없거나 기본 세션에서만 가능한 작업, 위임이 도움이 되지 않는 경우, 안전·개인정보 보호·동시 쓰기 제약이 있는 경우에만 직접 처리합니다.

어려운 질문은 기본적으로 기본 세션의 Luna Max가 직접 결정합니다. 에스컬레이션이 필요하면 Astra를 명시적으로 선택할 수 있습니다. 미리 정의된 Luna 역할은 리프 역할이며 하위 에이전트를 생성하지 않습니다.

---

## 1. `Luna_1M_Ultra` 프로필의 전역 `config.toml`

다음 내용을 `~/.codex/config.toml` 또는 현재 사용 중인 Codex 설정에 병합합니다.

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

`max_concurrent_threads_per_session = 12`는 동시에 사용할 수 있는 작업자 스레드의 최대 용량이지 목표 개수가 아닙니다. 이 설정만으로 에이전트가 시작되지는 않습니다. 위임 시점은 작업 라우팅에 따라 결정됩니다.

참고:

- `max_depth`는 이전 V1 멀티 에이전트 동작에 적용됩니다. V2에서는 무시될 수 있습니다.
- `expose_spawn_agent_model_overrides = true`는 오케스트레이터에 `spawn_agent` 모델 재정의 제어 항목을 표시합니다. 이는 설정 선택지를 노출하는 것이며, 숨겨진 추론 과정을 노출하지 않습니다.
- 전역 컨텍스트 및 압축 요청은 GPT-6 Luna 기본 세션과 네 가지 표준 Luna 역할 모두에 적용됩니다.
- GPT-6 Luna의 공식 컨텍스트 윈도우는 1.05M입니다. 설치된 로컬 모델 카탈로그는 현재 이를 `max_context_window = 872_000`으로 제한하며, 런타임에서 적용되는 압축 한도는 `828_400`입니다.
- 프로필 이름은 모델 ID를 대체하지 않습니다. 최대 추론 강도와 함께 `gpt-6-luna`를 사용하세요.
- 데스크톱 추론 강도 사용 가능 목록에는 `max`와 `ultra`가 포함됩니다. 정확한 값은 `enabled-reasoning-efforts = ["low", "medium", "high", "xhigh", "persistent", "max", "ultra"]`입니다.
- 모델 선택기에서 `max`와 `ultra`를 모두 활성화된 상태로 둡니다. GPT-6 Luna는 `max`를 지원하지만 `ultra`는 지원하지 않으므로 이 프로필에는 `max`를 사용하세요.
- 추론 강도는 모델 ID와 별개의 설정입니다. 모델 필드에는 `gpt-6-luna`를 사용하세요.
- TUI 푸터는 `status_line = ["model-with-reasoning", "context-remaining", "current-dir"]`를 사용합니다. `context-remaining`은 컨텍스트 사용량 표시를 활성화합니다.
- `[desktop].show-context-window-usage = true`는 앱 작성창의 컨텍스트 윈도우 사용량 표시기를 활성화합니다. 이는 터미널 TUI 푸터 설정과 별개입니다.
- `followUpQueueMode = "queue"`이면 현재 실행이 끝날 때까지 후속 프롬프트를 대기시킵니다. `steer`이면 활성 실행에 프롬프트를 적용합니다. 정확한 값 `queue`를 사용하세요. `stack`은 지원되지 않습니다.

### 앱 컨텍스트 표시기 문제 해결

- `[desktop].show-context-window-usage = true`는 작성창의 컨텍스트 표시기를 활성화합니다. `tui.status_line`은 터미널 TUI 푸터만 설정합니다.
- 현재 데스크톱 빌드는 세션에 컨텍스트 사용률이 있고 반응형 작성창 레이아웃에 표시 공간이 있을 때 표시기를 렌더링합니다. 창이 좁거나 사이드바가 펼쳐져 있으면 표시기가 숨겨질 수 있습니다. 창을 넓히거나 사이드바를 접으세요.
- 이 설정에서는 설정 > 일반 > 컨텍스트 길이 사용량이 활성화되어 있습니다. 작성창에 공간이 있는데도 표시기가 보이지 않는다면 활성 스레드에서 사용량 메타데이터를 제공하지 않을 수 있습니다.

---

## 2. `Luna_1M_Ultra` 라우팅 프로필의 Luna 역할

`~/.codex/agents/` 아래에 다음 파일을 만듭니다.

이 네 가지 역할 파일에서는 `model_context_window`와 `model_auto_compact_token_limit`을 의도적으로 생략합니다. 표준 역할은 모두 1절의 전역 기본값을 상속합니다. 앞서 설명한 것처럼 로컬 모델 카탈로그가 상속된 요청값을 제한할 수 있습니다. 각 파일은 `gpt-6-luna`를 사용하며, `Luna_1M_Ultra`는 개념적인 프로필 이름일 뿐입니다.

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

## 3. 표준 `Luna_1M_Ultra` 역할 집합

활성 `~/.codex/agents/` 디렉터리에는 다음 네 가지 표준 Luna 역할만 있습니다.

| 역할 | 추론 강도 | 컨텍스트 | 권한 |
|---|---:|---:|---|
| `luna_medium` | medium | 전역 기본값 | read-only |
| `luna_high` | high | 전역 기본값 | workspace-write |
| `luna_max` | max | 전역 기본값 | workspace-write |
| `luna_review` | xhigh | 전역 기본값 | read-only |

특정 호출자가 여전히 필요로 하지 않는 한 활성 역할 디렉터리에 호환성 별칭을 추가하지 마세요.

---

## 4. `Luna_1M_Ultra`용 `AGENTS.md` 라우팅 블록

이 블록을 활성 전역 또는 프로젝트 `AGENTS.md`에 추가합니다.

```markdown
<!-- BEGIN ASTRA_LUNA_1M -->
## Luna Max 라우팅

Luna Max는 기본 세션이며 오케스트레이션, 어려운 기술적 의사결정, 통합, 최종 승인을 담당합니다.
위임되는 모든 AI 작업에는 Luna를 사용합니다. Astra는 선택적으로 명시해 사용하는 에스컬레이션 경로이며 기본 모델이 아닙니다. Sol, Terra 또는 Astra 하위 세션은 사용하지 않습니다.

모든 사소하지 않은 작업에서(저장소 작업에만 국한되지 않음) 요청을 선제적으로 나누고 유용한 독립 작업 흐름을 Luna 작업자에게 위임합니다. 입력이 명확해지는 대로 독립 조사, 구현 단위, 소스 확인, 테스트, 검토를 병렬로 시작합니다. 주요 작업 흐름이 하나뿐인 경우에도 실질적인 가치가 있다면 별도의 분석 또는 검증 단위를 위임합니다. 사소한 단일 단계 요청, 본질적으로 분리할 수 없거나 기본 컨텍스트에서만 가능한 작업, 위임이 도움이 되지 않는 경우, 안전·개인정보 보호·동시 쓰기 제약이 있는 경우에만 직접 처리합니다. 의존 작업은 순차적으로 진행합니다. 작업자들이 일하는 동안 Luna Max는 같은 범위를 반복하지 않으며 오케스트레이션, 통합, 최종 승인 책임을 계속 맡습니다.

실질적인 작업에서는 기본적으로 최대 추론 강도의 `luna_max`를 사용합니다. 범위, 작업량 또는 권한이 더 잘 맞으면 `luna_medium`, `luna_high` 또는 `luna_review`를 바로 선택합니다. 사용자가 역할을 명시적으로 요청하도록 요구하지 않습니다.

설정된 최대 12개 스레드 한도 내에서 실질적으로 유용한 독립 작업 흐름 수만큼 에이전트를 사용합니다. 여섯 개를 목표로 삼거나 임의로 자리를 채우지 않습니다. 이 한도는 목표 개수가 아니며, 설정만으로 에이전트가 시작되지는 않습니다. 유용한 작업자를 재사용하고, 중복 범위와 겹치는 편집을 피하며, 의존 작업은 순차적으로 진행합니다. Luna 작업자는 리프 역할이며 하위 에이전트를 생성하지 않습니다.

작업 난이도와 권한 요구사항에 따라 Luna 역할을 직접 선택합니다.
- `luna_medium`: medium — 범위가 제한된 흐름/로그/의존성 분석.
- `luna_high`: high — 명확한 소규모 수정과 일상적인 테스트.
- `luna_max`: max — 실질적인 구현과 어려운 디버깅.
- `luna_review`: xhigh — 읽기 전용 권한으로 독립적인 심층 검토.

역할을 필수 추론 단계처럼 순서대로 실행하지 않습니다.
`luna_max` 작업자는 구현에 필요한 일반적인 검색을 직접 수행합니다.
Luna 작업자는 리프 역할이며 하위 에이전트를 생성하지 않습니다.

관련된 조사, 구현, 테스트, 수정에는 동일한 Luna 작업자를 재사용하는 편이 좋습니다.
하나의 일관된 작업을 진행하는 단계 사이에 유용한 작업자를 종료하지 않습니다.
관련 없는 작업이나 일회성 작업에는 필요한 작업 컨텍스트만 전달해 새 작업자를 시작합니다.
전역 기본 컨텍스트 윈도우가 사용 가능하다는 이유만으로 모두 채우지 않습니다.

spawn API에서 이력 제어를 제공한다면 의도에 맞게 선택합니다.
- V2: `fork_turns = "none" | "all" | "<positive integer>"`
- V1: `fork_context = false | true`
하나의 spawn 호출에서 두 형식을 함께 보내지 않습니다.

실질적으로 다른 두 가지 접근 방식이 진전 없이 실패하면, Luna는 해결되지 않은 정확한 질문을 Luna Max에 보고합니다.
기본적으로 Luna Max가 직접 결정한 뒤 동일한 Luna 작업자가 구현과 검증을 이어갑니다. 명시적으로 선택한 Astra 에스컬레이션은 요청된 경우에만 결정 과정에 참여할 수 있습니다.
보안 또는 데이터 손실 위험은 즉시 에스컬레이션합니다.

Luna Max가 직접 처리할 수 있는 경우는 사소한 단일 단계 요청, 본질적으로 분리할 수 없거나 기본 컨텍스트에서만 가능한 작업, 위임이 도움이 되지 않는 경우, 안전·개인정보 보호·동시 쓰기 제약이 있는 경우뿐입니다. Luna Max도 처리할 수 있다는 이유만으로 유용한 위임을 생략하지 않습니다.

설정된 최대 12개 스레드 한도 내에서 작업이 뒷받침하는 만큼 유용하고 독립적인 작업자를 사용합니다. 이 한도는 목표 개수가 아니며 여섯 개를 채울 필요도 없습니다.
중복 범위를 만들거나 편집이 서로 겹치게 하지 않습니다.
독립 검토가 정확성을 실질적으로 높이는 경우에 사용하고, 형식적인 검토자를 두지 않습니다.
실행하지 않은 테스트가 통과했다고 주장하지 않습니다.

GitHub 라우팅:
- 중요 저장소 및 GitHub 작업에도 동일한 선제적 위임 정책을 적용합니다. 실질적인 검색, diff 분석, 코드 또는 문서 변경, 테스트, GitHub CLI 준비, 이슈/PR 초안 작성에는 기본적으로 `luna_max`를 사용합니다. 범위와 권한이 더 잘 맞으면 다른 Luna 역할을 바로 선택하고, 독립 검토에는 `luna_review`를 사용합니다.
- 명시적으로 요청된 어려운 의사결정 또는 공개 저장소 작업에 필요한 수준으로 선택적 Astra 에스컬레이션을 제한합니다. 기본 담당자는 Luna Max입니다.
- 비밀 정보, 로컬 설정, 자격 증명 또는 검토되지 않은 백로그를 공개하지 않습니다.
<!-- END ASTRA_LUNA_1M -->
```

---

## 5. `Luna_1M_Ultra` 컨텍스트 전략

이 설정은 모든 Luna 작업이 빈 컨텍스트에서 시작하도록 강제하지 않습니다.

권장 동작:

| 상황 | 컨텍스트 전략 |
|---|---|
| 사소한 단일 단계 조회 | 기본 세션에서 직접 처리합니다. 유용하다면 범위가 한정된 조사에 `luna_medium`을 사용합니다. |
| 조사 → 구현 → 수정으로 이어지는 의존 작업 | 동일한 Luna 작업자를 순차적으로 재사용합니다. |
| 독립 작업 흐름이 여러 개인 사소하지 않은 작업 | 설정된 최대 12개 스레드 한도 내에서 작업이 뒷받침하는 만큼의 유용한 독립 단위를 병렬로 시작합니다. |
| 그 외 단일 작업 흐름의 사소하지 않은 작업 | 가치가 있다면 유용한 독립 분석이나 검증 단위를 위임합니다. |
| 관련 없는 작업 | 새로운 작업자를 시작합니다. |
| 해결되지 않은 어려운 의사결정 | 문제와 근거를 기본 세션의 Luna Max에 보고합니다. 명시적으로 선택한 경우에만 Astra를 사용합니다. |

목표는 Luna 입력 토큰을 단순히 줄이는 것이 아니라 중복 추론과 재조사를 최소화하는 것입니다.

---

## 6. `Luna_1M_Ultra` 검증

설정을 적용한 후 다음을 확인합니다.

1. 기본 세션이 최대 추론 강도의 `gpt-6-luna`를 사용하며, 전역 기본값 `model_context_window = 1_000_000` 및 `model_auto_compact_token_limit = 900_000`을 요청하는지 확인합니다.
2. 각 표준 Luna 역할이 역할별 재정의 없이 해당 전역 컨텍스트 및 압축 설정을 상속하는지 확인합니다.
3. `luna_max`가 `gpt-6-luna`/max로 결정되는지 확인합니다.
4. `luna_review`가 `gpt-6-luna`/xhigh로 결정되는지 확인합니다.
5. GPT-6 Luna의 공식 컨텍스트 윈도우가 1.05M인지, 설치된 로컬 카탈로그가 `max_context_window = 872_000` 및 런타임에서 적용되는 압축 한도 `828_400`을 보고하는지 확인합니다.
6. 기존 프로젝트/프로필 재정의가 모델 또는 추론 설정을 조용히 바꾸지 않는지 확인합니다.
7. 라우팅 설정에서 Sol/Terra 및 Astra 하위 세션을 기본 선택하지 않는지 확인합니다.
8. `Luna_1M_Ultra`가 개념적인 프롬프트/라우팅 별칭으로 유지되고 `model` 값으로 사용되지 않는지 확인합니다.
9. 데스크톱 추론 강도 사용 가능 목록에 `max`와 `ultra`가 포함되는지 확인합니다.
10. GPT-6 Luna에는 `max`와 `xhigh`를 사용하고, 이 모델이 지원하지 않는 `ultra`는 선택하지 않는지 확인합니다.
11. TUI 푸터에 `context-remaining` 항목이 포함되는지 확인합니다.
12. 데스크톱 후속 프롬프트 동작이 `queue`로 설정되는지 확인합니다.
13. 앱 작성창의 컨텍스트 윈도우 사용량 표시기가 활성화되어 있는지 확인합니다.

모델이 스스로 보고한 정체 정보만으로 검증하지 마세요. 가능한 경우 실제 세션/모델 메타데이터를 우선 사용합니다.

---

## 7. `Luna_1M_Ultra` 참고 사항

- 전역 설정은 GPT-6 Luna에 `model_context_window = 1_000_000` 및 `model_auto_compact_token_limit = 900_000`을 요청하며, 미리 정의된 네 역할 모두 이 설정을 상속합니다.
- GPT-6 Luna의 공식 컨텍스트 윈도우는 1.05M입니다. 설치된 로컬 모델 카탈로그는 현재 이를 `max_context_window = 872_000`으로 제한하며, 런타임에서 적용되는 압축 한도는 `828_400`입니다.
- 컨텍스트 값을 요청해도 모든 Codex 빌드/계정에서 전체 용량을 사용할 수 있는 것은 아닙니다.
- 프롬프트/캐시 재사용은 조건부이므로 스레드를 재사용한다고 해서 자동으로 된다고 가정해서는 안 됩니다.
- `AGENTS.md`는 라우팅 지침이지 강제 모델 허용 목록이나 보안 경계가 아닙니다.
- 별도로 의도해 관리하지 않는 한 권한, 제공자 설정, MCP 구성 및 관련 없는 프로젝트 설정은 변경하지 마세요.

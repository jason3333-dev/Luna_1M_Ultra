# Luna_1M_Ultra v2 — GPT-6 Luna 마이그레이션

날짜: 2026-09-23

## v2 기본 구성

- 메인 모델: `gpt-6-luna` / `max`
- 기본 위임 모델: `gpt-6-luna` / `max`
- 1M 컨텍스트 요청, 자동 압축 기준 `900,000`
- 사전정의 역할 4개(`luna_medium`, `luna_high`, `luna_max`, `luna_review`)를 메인 모델이 필요할 때 호출
- 역할 effort: `luna_medium` medium, `luna_high` high, `luna_review` xhigh, `luna_max` max
- 스크린샷의 6개 항목은 병렬 작업 사례이며 에이전트 역할의 종류를 뜻하지 않음
- GPT-6 Luna 공식 컨텍스트 창은 1.05M이지만 현재 설치된 Codex 카탈로그는 872K로 제한

## 활성화된 설정

- 기본 reasoning: `max`; `ultra`는 선택기에 남겨두되 GPT-6 Luna에서는 지원하지 않음
- 앱 설정 > 일반 > 컨텍스트 길이 사용량 표시: 활성화 (`[desktop].show-context-window-usage = true`)
- 앱 composer는 세션 사용률 데이터가 있고 너비가 충분할 때 컨텍스트 게이지를 표시함
- 터미널 TUI 컨텍스트 표시: `context-remaining`
- Reasoning 선택기 목록에는 `max`, `ultra` 모두 포함
- 후속 입력 대기 설정의 유효한 값은 `queue`이며, `stack`은 지원되지 않음

## 오늘의 사용사례

README에 다음 6개 병렬 작업 사례와 스크린샷을 정리했습니다.

1. Core gate — planning-focused test
2. Operator review — halt/resume race analysis
3. Fill execution p1 — delivery identity logic
4. Accounting p1 — filename parsing analysis
5. Pair machine p1 — external increase impact
6. Candidate p1 — parent communication options

Pro 5X의 주간 사용량 `5분당 약 1%`는 제공된 실행 관찰값으로만 기록했으며, 공식 quota 보장값으로 해석하지 않습니다.

## 반영된 문서와 커밋

- `README.md` — 병렬 실행 사용사례 및 스크린샷
- `Luna_1M_Ultra_Setup.md` — GPT-6 Luna 모델, max reasoning 및 컨텍스트 표시 설정
- `parallel-luna-max-run.png` — 사용사례 스크린샷
- GPT-6 Luna 마이그레이션 커밋은 아직 생성되지 않음

Follow-up behavior: `queue` is configured, so new prompts wait for the active run to finish.

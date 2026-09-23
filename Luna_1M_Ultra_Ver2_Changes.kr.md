# Luna_1M_Ultra v2 — GPT-6 Luna 마이그레이션

[English](./Luna_1M_Ultra_Ver2_Changes.md) | [한국어](./Luna_1M_Ultra_Ver2_Changes.kr.md)

날짜: 2026-09-23

## 변경 사항

- 메인 세션과 기본 위임 작업자의 모델을 `gpt-5.6-luna`에서 `gpt-6-luna`로 변경하고, 둘 다 `max` 추론 수준으로 설정했습니다.
- `luna_low`를 제거하고 Luna 역할 네 개를 유지했습니다: `luna_medium`/medium, `luna_high`/high, `luna_max`/max, `luna_review`/xhigh.
- `model_context_window = 1_000_000`과 `model_auto_compact_token_limit = 900_000`을 요청했습니다. GPT-6 Luna의 공식 컨텍스트 창은 1.05M이었지만, 마이그레이션 당시 설치된 Codex 카탈로그에는 `max_context_window = 872_000`과 실제 런타임 압축 기준 `828_400`이 표시되었습니다.
- `[desktop].show-context-window-usage = true`로 앱의 컨텍스트 사용량 표시를, `context-remaining`으로 터미널 TUI 표시를 활성화했습니다.
- `followUpQueueMode`를 지원하지 않는 `stack`에서 `queue`로 바로잡았습니다. 새 프롬프트는 실행 중인 작업이 끝날 때까지 대기하도록 설정되었습니다.
- 추론 수준 선택기에 `max`와 `ultra`를 모두 남겼습니다. `gpt-6-luna`에는 `max`를 사용했고, 이 모델은 `ultra`를 지원하지 않았습니다.

## README 스크린샷과 사용량 관찰치

`README.md`에는 병렬 작업 여섯 개를 보여주는 스크린샷 `parallel-luna-max-run.png`을 기록했습니다.

1. Core gate — 계획 중심 테스트
2. Operator review — 중단/재개 경합 분석
3. Fill execution p1 — 전달 식별 정보 로직
4. Accounting p1 — 파일명 파싱 분석
5. Pair machine p1 — 외부 증가 영향
6. Candidate p1 — 상위 작업과의 소통 방안

제공된 Pro 5X 관찰치는 이런 병렬 실행에서 5분당 주간 사용량의 약 1%였습니다. 이는 계획을 세울 때 참고하는 관찰값이며, 공식 할당량·청구 약속·보장된 사용률은 아닙니다.

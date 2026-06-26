# Design Basis — 왜 이 시스템이 이렇게 생겼나

> **로드 정책**: 이 파일은 평소 작업에서 읽지 않는다. 시스템 파일(`AGENTS.md`, `_shared/*`, `_templates/*`)을 수정·검증할 때만 읽는다.

## 0. 출처

- 원본 starter: multi-agent-starter
- Codex flavor: multi-agent-starter의 Codex orchestrator 파생본
- 4원칙(Operating Principles) 출처: https://github.com/multica-ai/andrej-karpathy-skills (MIT 선언, LICENSE 파일 부재 — 표기는 `NOTICE` 참조)
- 사용자 결정: Codex가 메인 오케스트레이터가 되며, worker pool은 콘텐츠/SEO 운영팀(`claude-main`, `codex-main`, `codex-critic`, `gemini`)으로 둔다.

## 1. 핵심 개념 → 시스템 규칙 매핑

| 개념 | 시스템 규칙 | 주의 |
|------|-------------|------|
| 컨텍스트 = 유한 attention budget | context.md <= 1500자, brief <= 1200자 | 한도 변경 시 불변식 갱신 |
| Progressive disclosure | sources/ 경로 참조, brief 최소화 | 긴 자료 inline 금지 |
| Filesystem = memory | task/context/log/brief/result | 런타임 상태에 의존하지 않음 |
| Append-only + provenance | log.md append-only, 태그 6종 | 로그 삭제·수정 금지 |
| Never trust upstream | worker result 검증 후 채택 | critic/gemini 출력도 사실검증 |
| Adversarial review | `codex-critic` read-only 비평 | 정책·결함 차단 요소를 통과 전 해소 |
| 5인 운영팀 | Orchestrator + 4 workers | 팀은 고정, 호출은 최소 set |
| 최소 worker set | routing.md decision tree | 모든 worker 기본 호출 금지 |
| Fan-in 충돌 해소 | 출처 병기, 사실검증, `[DECISION]` | 다수결 금지 |
| Skill supply gate | 있음/부족/없음 판단 | 영구 등록은 사용자 승인 |

## 2. 권위 우선순위

`AGENTS.md` > `_shared/routing.md`·`approval-policy.md`·`orchestrator-rules.md` > `_templates/*`.

충돌 발견 시 낮은 쪽을 높은 쪽에 맞추고, 작업 중인 task의 `log.md`에 `[DECISION]`으로 남긴다.

## 3. 결정 기록

- **D1 write_scope 값 집합** = `none | tasks-only | "패턴"`. `tasks-only`는 `tasks/<task>/` 내부만 쓰는 기본값이다.
- **D2 worker pool** = Codex 버전은 콘텐츠/SEO 운영팀 기준 4역할을 사용한다. `claude-main`은 기획가, `codex-main`은 실행가, `codex-critic`은 read-only 비평가, `gemini`는 장문/시각 검토자다.
- **D2a 5인 운영팀** = 실제 운영 단위는 Codex Orchestrator 팀장 1명 + worker 4명이다. 사용자가 "멀티에이전트로 이 사이트..."라고 요청하면 이 팀 구성을 적용하되, worker 호출은 최소 set과 승인 게이트를 따른다.
- **D3 codex-critic 선행조건** = 리뷰 대상 산출물 경로가 존재해야 한다. 대상은 `claude-main`/`codex-main result.md`, Orchestrator 산출물·기존 코드·문서·소스도 가능하다.
- **D4 gemini 정책** = 백엔드 Antigravity `agy` CLI(`_shared/backends.json` 정본, 디스패처 `call_worker.sh`). 기본 `gemini-3.1-pro-high`, 빠른 경로 `gemini-3-flash`/`pro-low`, 폴백 `api`. 옛 `mcp__gemini-pro__*` 프록시 브리지 폐기. `pro-high` 제외 사유(옛 프록시 400)는 agy엔 비해당(2026-06-02 실증). agy 모델은 전역·계정단위라 gemini 전용 전역을 pro-high로 운용.
- **D5 Orchestrator** = Codex 현재 세션이 단일 Orchestrator다. 별도 long-lived supervisor worker나 worker 재귀 위임 계층은 쓰지 않는다.
- **D6 모델 식별자 표기** = Codex와 Claude는 환경 설정/별칭을 따르고 repo에 버전 문자열을 핀하지 않는다. Gemini는 백엔드 `agy` CLI·기본 `gemini-3.1-pro-high`를 `backends.json`에 명시 핀(agy 모델이 전역·계정단위라 per-call 핀 불가). 세부는 D4.
- **D7 카파시 4원칙 층별 적용** = 오케스트레이터 지침(AGENTS.md "Operating Principles" 섹션) 풀버전 verbatim 차용 / 워커층 유일 정본은 `_templates/worker-brief.md`의 "Worker 행동 규약" 고정 블록 — ②단순함·③외과수술식 그대로 + ①추측전질문은 번역형(워커는 one-shot/headless라 사용자 질문 채널 없음 → 가정 명시·불확실/불일치를 result.md Issues/Caveats에 표면화) / ④목표기반 loop은 오케스트레이터 전용(Verification Checklist 루프와 결합). 워커 brief에 "사용자에게 질문" 지시 금지. 출처: multica-ai/andrej-karpathy-skills (MIT 선언, LICENSE 파일 부재 — `NOTICE` 정본, 2026-06-10 확인).
- **D8 스킬 수급 게이트** = Orchestrator가 작업 전 필요한 스킬을 `있음(사용) / 부족(개선) / 없음(개발)`으로 판단한다. 보유 후보는 `ait-builder`, `blog-optimizer`, `title-generator`, `adsense-optimizer`이며, 신규·개선 스킬은 `codex-critic` 검증 후 사용자 승인 없이는 영구 등록하지 않는다.
- **D9 승인 게이트** = 초안·검수·진단은 자율이다. 게시/배포, 삭제, 결제, 스킬 풀 영구 등록, 시스템 설정·헌법·정책 변경은 실행 직전 한 지점에서 사용자 승인을 받는다.

## 4. 불변식

구체 항목과 점검 명령은 `_shared/system-invariants.md`에 둔다.

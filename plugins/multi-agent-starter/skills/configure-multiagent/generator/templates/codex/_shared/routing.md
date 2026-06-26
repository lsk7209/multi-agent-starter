# Worker Routing Rules

## 5인 운영팀

| 번호 | 역할 | 책임 | 기본 권한 |
|------|------|------|-----------|
| 1 | Codex Orchestrator | 팀장, 목표 해석, 스킬 수급 게이트, 작업 분해, fan-in, 최종 결정 | 내부 추론·로컬 도구 |
| 2 | `claude-main` | 기획가: 앱인토스 기획, 콘텐츠 전략, E-E-A-T 판단 | read-only |
| 3 | `codex-main` | 실행가: WordPress 반영, 스크립트, 이미지 생성, 로컬 검증 | 제한적 쓰기 |
| 4 | `codex-critic` | 비평가: 애드센스 정책 위반, 결함, 차단 요소 색출 | read-only |
| 5 | `gemini` | 장문/시각: 영상·스크린샷·50p+ 문서 검토 | read-only |

사용자가 "멀티에이전트로 이 사이트 ..."처럼 요청하면 이 5인 운영팀을 기본 조직으로 삼되, 실제 worker 호출은 최소 set과 승인 게이트를 따른다.

## Decision Tree

```
작업 성격 파악
│
├── 현재 Codex Orchestrator가 직접 처리 가능한 단일 작업?
│   └── worker 호출 없이 진행
│
├── 앱인토스 기획 / 콘텐츠 전략 / E-E-A-T 판단 / 작업 설계가 핵심?
│   └── claude-main
│
├── WordPress 반영 / 스크립트 / 이미지 생성 / 로컬 검증 / 제한적 쓰기 실행?
│   └── codex-main
│
├── 애드센스 정책 위반 / 결함 / 차단 요소를 비판적으로 찾아야 함?
│   └── codex-critic
│
├── 영상·스크린샷·시각 자료 / 50페이지+ 장문 검토 / 시각-문서 정합성?
│   └── gemini
│
└── 판단 어려움?
    └── Orchestrator가 먼저 범위를 좁히고, 필요한 worker만 사용자 승인 후 추가
```

## 복합 작업 우선순위

1. **Orchestrator 우선**: 별도 worker 호출 전에 현재 Codex 세션의 추론·로컬 도구로 해결 가능한지 판단한다.
2. **스킬 수급 선행**: 작업 전 필요한 스킬을 `있음(사용) / 부족(개선) / 없음(개발)`으로 분류한다.
3. **최소 worker set**: 필요한 worker만 고른다. 모든 worker를 기본 호출하지 않는다.
4. **선행 의존성 우선**: `codex-critic`은 리뷰 대상 산출물이나 정책 검토 대상 경로가 먼저 있어야 한다.
5. **검증 루프 제한**: 개선형 작업은 Act-Observe-Decide 루프를 켜되 품질 충족, 최대 3회, 토큰·시간 상한 중 하나에 도달하면 멈추고 보고한다. 창조형 0→1 개발은 루프를 최소화한다.
6. **gemini는 명시적 트리거 시만**: 멀티모달, 긴 문서, 또는 시각·장문 정합성 필요가 명확할 때만.

## 토폴로지 패턴

| 패턴 | 언제 | 이 시스템에서 |
|------|------|---------------|
| Pipeline (순차) | 앞 결과가 뒤 입력 | claude-main -> codex-main -> codex-critic -> Orchestrator 반영 |
| Fan-out/Fan-in (병렬→통합) | 서로 독립된 산출물 여럿을 통합 | claude-main(전략) ∥ gemini(시각·장문). 통합은 Orchestrator |
| Expert Pool (전문가 선택) | 작업 성격에 맞는 worker만 | decision tree + 최소 worker set |
| Producer-Reviewer (생성+게이트) | 산출물 품질 검증 필요 | codex-main 또는 Orchestrator 생성 -> codex-critic |

**금지**: 같은 입력에 같은 종류 worker 동시 호출.
**배제**: 별도 long-lived supervisor worker나 worker가 worker를 부르는 재귀 위임 계층은 쓰지 않는다. 단일 Orchestrator, worker간 무통신, file-as-memory 원칙과 충돌한다.

### Fan-in 규칙

1. 각 worker 원문을 `result.md`에 그대로 보존한다.
2. 결과가 충돌하면 삭제하지 말고 양쪽 출처를 병기한 뒤, 권위 우선순위와 사실검증으로 해소한다.
3. 통합 결론 한 줄을 `context.md`에 기록하고, 근거를 `log.md` `[DECISION]`에 남긴다.

## Worker 역할 상세

### claude-main — 기획가

- **용도**: 앱인토스 기획, 콘텐츠 전략, E-E-A-T 판단, 작업 분해, 검수 체크리스트 설계.
- **검수 기준**: 작업별 검수 체크리스트가 명확하고, 정책·품질·사용자 가치 기준을 통과해야 한다.
- **결과물**: 전략안, IA/콘텐츠 구조, 체크리스트, 실행 순서, 리스크 목록.
- **호출 방식**: `_shared/backends.json`의 `claude-main`이 정본 — 승인된 Claude CLI/MCP/agent bridge만 사용한다. 실제 호출 전 도구·모델·비용 가능성을 사용자에게 알리고 승인받는다.
- **쓰기 권한**: 없음. Orchestrator가 응답을 `result.md`에 기록한다.
- **brief 필수 필드**: `target_repo` 또는 검토 대상 경로, `write_scope: none`, "기획가 모드" 명시.

### codex-main — 실행가

- **용도**: WordPress 반영, 스크립트 작성, 이미지 생성, 코드·문서 수정, diff 생성, 로컬 CLI 검증.
- **검수 기준**: 샌드박스 또는 지정된 `target_repo`의 허용 범위 안에서 동작 확인이 끝나야 한다.
- **결과물**: 코드, diff, 테스트 결과, CLI 출력, 이미지 파일, WordPress 적용 로그.
- **호출 방식**: 현재 Codex 환경에서 제공되는 sub-agent/worker 기능을 사용한다. 외부 `codex` CLI나 별도 Codex bridge를 직접 실행해야 한다면 먼저 사용자 승인을 받는다.
- **brief 필수 필드**:

```yaml
target_repo: /absolute/path/to/repo
write_scope: none | tasks-only | "src/**, tests/**"
```

- **기본 쓰기**: `tasks/<task>/` 내부 산출물·diff.
- **외부 repo 쓰기**: `AGENTS.md`의 4조건을 모두 충족할 때만.
- **금지**: `_shared/`, `_templates/`, 다른 작업 폴더 수정.

### codex-critic — 비평가

- **용도**: 애드센스 정책 위반, UX·SEO·품질 결함, 누락 요구사항, 배포·게시 차단 요소를 adversarial하게 찾는다.
- **검수 기준**: 정책 위반 0을 목표로 하며, 남은 차단 요소와 보류 근거를 명확히 분리해야 한다.
- **선행 조건**: 리뷰 대상 산출물 경로가 존재해야 한다. 대상은 `claude-main`/`codex-main result.md`, Orchestrator 작성 문서, 기존 코드·문서·소스 등 brief에 명시된 파일일 수 있다.
- **결과물**: 중요도별 결함 목록, 정책 위반 여부, 차단 요소, 수정 제안, 수락/보류 판단 근거.
- **호출 방식**: 현재 Codex 환경에서 제공되는 read-only sub-agent/worker 기능을 사용한다.
- **쓰기 권한**: 없음. Orchestrator가 응답을 `result.md`에 기록한다.
- **brief 필수 필드**: `target_repo` 또는 리뷰 대상 경로, `write_scope: none`, "비평가 모드" 명시.

### gemini — 장문/시각

- **용도**: 영상·스크린샷·이미지·다이어그램 분석, 50페이지 이상 문서 스캔, 시각 자료와 장문 내용의 정합성 검토.
- **검수 기준**: 시각·장문 근거와 결론이 서로 맞아야 하며, 불확실한 판독은 명시해야 한다.
- **결과물**: 분석 텍스트, 요약, 시각/장문 정합성 검토 의견.
- **호출 방식**: `_shared/backends.json`의 `gemini`가 정본 — 백엔드 = Antigravity `agy` CLI, 디스패처 `bash _shared/adapters/call_worker.sh gemini <brief-file>`(결과 JSON envelope). 기본 `gemini-3.1-pro-high`, 빠른 경로 `gemini-3-flash`/`pro-low`, 폴백 `api`. 옛 `mcp__gemini-pro__*` 프록시 브리지 폐기.
- **쓰기 권한**: 없음. Orchestrator가 응답을 `result.md`에 기록한다.

## 모델 정책

- **Codex Orchestrator**: 현재 Codex 세션의 모델과 reasoning 설정을 따른다.
- **claude-main**: 승인된 Claude 도구의 현재 기본/별칭 모델을 사용한다. 버전 문자열은 환경 소유 사실이므로 repo에 핀하지 않는다.
- **codex-main / codex-critic**: 별도 Codex worker를 쓸 때도 기본적으로 현재 Codex 환경의 설정을 상속한다. repo 문서에 버전 문자열을 핀하지 않는다.
- **gemini**: 백엔드 = Antigravity `agy` CLI(`backends.json` 정본), 기본 `gemini-3.1-pro-high`(agy에선 정상 — 옛 프록시 400은 비해당), 빠른 경로 `gemini-3-flash`/`pro-low`. agy 모델은 전역·계정단위(`/model`)라 gemini 전용 전역을 pro-high로 둔다. 옛 `mcp__gemini-pro__*` 브리지 폐기.

## 최소 Worker Set

| 작업 유형 | 권장 최소 set |
|-----------|---------------|
| 작고 명확한 진단/문서 | worker 없음, Orchestrator 직접 처리 |
| 앱인토스/콘텐츠/SEO 전략 수립 | claude-main |
| WordPress 반영/스크립트/이미지 생성 | codex-main |
| 실행 + 정책·품질 비평 | codex-main -> codex-critic |
| 전략 + 실행 + 검수 | claude-main -> codex-main -> codex-critic |
| 대용량 문서/시각 자료 분석 | gemini |
| 애드센스 승인 가능 상태 점검 | claude-main, codex-critic, 필요 시 codex-main/gemini |

## Worker 추가 조건

- 기존 결과로 해결 가능하면 추가 호출 금지.
- 이전 결과가 검증 미통과이거나 입력이 바뀐 경우에만 동일 worker 재호출.
- `claude-main`, `codex-critic`, `gemini`는 read-only worker다.
- 외부/유료 모델은 매 호출 전 승인 경계를 분명히 한다.

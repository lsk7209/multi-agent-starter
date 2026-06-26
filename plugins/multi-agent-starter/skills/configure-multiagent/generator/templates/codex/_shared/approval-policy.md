# Worker Approval Policy

## 원칙

**모든 worker 호출은 작업별로 명시적 승인 필요** (`claude-main`, `codex-main`, `codex-critic`, `gemini` 전체 pool 적용).
`task.md`의 `workers_approved` 리스트에 없으면 호출 금지.

**예외**: Codex Orchestrator의 내부 추론은 worker 호출이 아니므로 승인 불필요. 다만 별도 `claude-main`, `codex-main`, `codex-critic`, `gemini` worker 호출은 승인 대상이다.

외부/유료 모델(`claude`, `gemini`, `openai`, `llm`, MCP/agent bridge 등)은 worker 승인과 별도로, 실제 호출 직전에 도구·모델·비용 가능성을 밝히고 사용자 승인을 받아야 한다.

## 승인 절차

1. Orchestrator가 worker 필요성을 판단한다 (`_shared/routing.md` 참조).
2. 사용자에게 다음 정보를 알려 승인 요청:
   - 어떤 worker를
   - 무슨 목적으로
   - 예상 호출 횟수
   - 외부/유료 모델이면 비용·쿼터 가능성
3. 승인 시 `task.md`의 `workers_approved`에 추가.
4. `log.md`에 `[APPROVAL]` 태그로 승인 기록.
5. 같은 작업에서 같은 승인 범위 안의 재호출은 재승인 불필요.

## 비가역 승인 게이트

초안 작성, 검수, 진단, 로컬 검증은 자율 진행한다. 다음 비가역 행동은 실행 직전 한 지점에서 명시적 사용자 승인을 받아야 한다.

- 게시/배포
- 삭제
- 결제/구매
- 스킬 풀 영구 등록
- 시스템 설정·헌법·정책 변경

스킬 사용 자체는 자율이다. 신규·개선 스킬을 영구 등록하려면 `codex-critic` 검증을 먼저 통과하고, 사용자 승인을 별도로 받아야 한다.

## 승인 예외

- **Orchestrator 내부 추론**: worker 호출이 아니므로 승인 불필요.
- **동일 작업 재호출**: `workers_approved`에 이미 있고 목적·write_scope가 같으면 재승인 불필요.
- **검증 실패 후 재시도**: 승인된 worker 범위 안에서 1회 자동 허용.
- **외부 쓰기 범위 변경**: `target_repo` 또는 `write_scope`가 바뀌면 기존 승인은 무효.

## 비용·쿼터 가이드라인

| Worker | 예상 비용 | 쿼터 부담 |
|--------|-----------|-----------|
| claude-main | 중간 | Claude API/구독 쿼터 |
| codex-main | 중간 | Codex 호출 쿼터 |
| codex-critic | 낮음-중간 | Codex 호출 쿼터 |
| gemini flash | 낮음 | Gemini 쿼터 |
| gemini pro | 중간-높음 | Gemini 쿼터 |

## 승인 기록 형식

```yaml
workers_approved:
  - worker: claude-main
    approved_at: <YYYY-MM-DD>
    purpose: 콘텐츠 전략 및 E-E-A-T 판단
    approved_by: user
  - worker: codex-main
    approved_at: <YYYY-MM-DD>
    purpose: WordPress 반영 및 로컬 검증
    approved_by: user
  - worker: codex-critic
    approved_at: <YYYY-MM-DD>
    purpose: 애드센스 정책·결함 비평
    approved_by: user
```

날짜 명령어: `date +%Y-%m-%d`

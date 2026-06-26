# System Invariants — 시스템 수정 후 자가 점검

> **로드 정책**: 평소 미로드. 시스템 파일 수정·검증 작업일 때만 사용한다.

## 불변식 목록

| ID | 불변식 |
|----|--------|
| INV1 | `write_scope` 값 집합이 `AGENTS.md`, `routing.md`, `worker-brief.md`, `task-folder.md`에서 동일 |
| INV2 | 콘텐츠/SEO worker pool 4역할(`claude-main`, `codex-main`, `codex-critic`, `gemini`)이 `AGENTS.md`, `routing.md`, `backends.json`에 존재 |
| INV3 | read-only worker(`claude-main`, `codex-critic`, `gemini`)가 외부 repo 쓰기 권한을 갖지 않음 |
| INV4 | log 태그가 정확히 `DECISION | WORKER_CALL | VERIFICATION | ERROR | APPROVAL | COMPLETE` 6종 |
| INV5 | context 한도 1500자, brief 한도 1200자가 정본 문서와 템플릿에서 일치 |
| INV6 | 권위 우선순위가 `AGENTS.md` 기준으로 기록됨 |
| INV7 | 재진입 프로토콜이 `orchestrator-rules.md`와 `AGENTS.md` 포인터에 모두 존재 |
| INV8 | 토폴로지 4패턴(Pipeline, Fan-out/Fan-in, Expert Pool, Producer-Reviewer)이 routing에 존재 |
| INV9 | gemini 백엔드가 `_shared/backends.json`에서 `agy` CLI(command agy)이고 기본 모델 `gemini-3.1-pro-high`; routing.md·D4가 backends를 정본 참조, 옛 `mcp__gemini-pro__*` 활성호출 없음 |
| INV10 | 스킬 수급 게이트가 `있음(사용) / 부족(개선) / 없음(개발)` 판단, `codex-critic` 검증, 사용자 승인 후 영구 등록을 명시 |
| INV11 | 카파시 4원칙(D7): `AGENTS.md`에 "Operating Principles" 섹션 존재, `_templates/worker-brief.md`에 "Worker 행동 규약" 고정 블록 존재, 블록 안에 사용자질문 지시(질문/ask) 없음, `worker-result.md` 체크리스트에 표면화 항목 존재 |

## 자가 점검 스크립트

`<설치한-폴더>`에서 실행한다.

```bash
ROOT=<설치한-폴더>

echo "INV1 tasks-only 분포"
grep -l 'tasks-only' "$ROOT/AGENTS.md" "$ROOT/_shared/routing.md" \
  "$ROOT/_templates/worker-brief.md" "$ROOT/_templates/task-folder.md"

echo "INV2 콘텐츠/SEO worker pool 4역할"
for w in claude-main codex-main codex-critic gemini; do
  grep -q "$w" "$ROOT/AGENTS.md" && echo " AGENTS $w PASS" || echo " AGENTS $w FAIL"
  grep -q "$w" "$ROOT/_shared/routing.md" && echo " routing $w PASS" || echo " routing $w FAIL"
  grep -q "\"$w\"" "$ROOT/_shared/backends.json" && echo " backends $w PASS" || echo " backends $w FAIL"
done

echo "INV3 read-only worker 쓰기 권한"
grep -n '"write_policy": "none"' "$ROOT/_shared/backends.json"
grep -n 'claude-main.*Never\|codex-critic.*Never\|gemini.*Never' "$ROOT/AGENTS.md" || true

echo "INV4 log 태그"
grep -n 'DECISION | WORKER_CALL | VERIFICATION | ERROR | APPROVAL | COMPLETE' "$ROOT/_templates/log.md" "$ROOT/AGENTS.md"

echo "INV5 한도 수치"
grep -rn '1500자\|1200자\|1500 chars\|1200 chars' "$ROOT/AGENTS.md" "$ROOT/_templates/context.md" "$ROOT/_templates/worker-brief.md"

echo "INV6 권위 우선순위"
grep -rn 'AGENTS.md.*routing.md' "$ROOT/_shared/design-basis.md" "$ROOT/_shared/orchestrator-rules.md"

echo "INV7 재진입"
grep -q '재진입 프로토콜' "$ROOT/_shared/orchestrator-rules.md" && echo " orchestrator-rules PASS" || echo " orchestrator-rules FAIL"
grep -q 're-entry protocol\|재진입 프로토콜' "$ROOT/AGENTS.md" && echo " AGENTS.md PASS" || echo " AGENTS.md FAIL"

echo "INV8 토폴로지 4패턴"
for p in 'Pipeline' 'Fan-out/Fan-in' 'Expert Pool' 'Producer-Reviewer'; do
  grep -q "$p" "$ROOT/_shared/routing.md" && echo " $p PASS" || echo " $p FAIL"
done

echo "INV9 gemini 백엔드 (backends.json agy·pro-high 둘 다 출력돼야 PASS)"
grep -n '"command": "agy"' "$ROOT/_shared/backends.json"
grep -n 'gemini-3.1-pro-high' "$ROOT/_shared/backends.json"
echo "INV9b 옛 프록시 활성호출 (출력 없어야 PASS; 폐기문맥 제외)"
grep -rn 'mcp__gemini-pro__\|mcp__gemini__gemini_' "$ROOT/_shared/routing.md" "$ROOT/_templates/task-folder.md" "$ROOT/AGENTS.md" | grep -viE '폐기|deprecat' || true

echo "INV10 스킬 수급 게이트"
grep -rn '있음(사용).*부족(개선).*없음(개발)\|스킬 수급 게이트\|영구 등록' "$ROOT/AGENTS.md" "$ROOT/_shared/orchestrator-rules.md" "$ROOT/_shared/approval-policy.md"

echo "INV11 카파시 4원칙 — Operating Principles 섹션 + Worker 행동 규약 블록 + result 표면화 항목 (셋 다 나와야 PASS)"
grep -n 'Operating Principles' "$ROOT/AGENTS.md"
grep -n 'Worker 행동 규약' "$ROOT/_templates/worker-brief.md"
grep -n '표면화' "$ROOT/_templates/worker-result.md"
echo "INV11b 블록 내 사용자질문 표현 (출력 없어야 PASS)"
sed -n '/^## Worker 행동 규약/,/^## Execution/p' "$ROOT/_templates/worker-brief.md" | grep -inE '질문|ask' || echo " 없음 PASS"
```

## 전면 재감사가 필요한 경우

- 새 외부 개념·레퍼런스를 시스템에 도입할 때
- worker pool 구성·역할이 바뀔 때
- 위 불변식으로 표현 불가한 구조 변경이 생길 때

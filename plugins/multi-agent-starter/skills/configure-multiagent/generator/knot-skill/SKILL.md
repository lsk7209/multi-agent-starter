---
name: knot
description: $KNOT_VAULT 지식 vault를 직접 다루는 능동 스킬 — verb 4개. save(자료를 inbox에 저장), ingest(prompts/ingest.md 규약으로 컴파일), query(prompts/query.md로 근거기반 질의), lint(scripts/lint.py + prompts/lint.md 건강검진). 트리거 예시 "knot에 저장해줘", "이거 vault에 넣어줘", "knot ingest", "지식그물 컴파일해줘", "knot에 물어봐", "vault에서 찾아줘", "knot lint", "vault 건강검진", "/knot". $KNOT_VAULT 미설정이면 안내만 하고 중단.
---

# knot — 지식 vault 능동 스킬

knot vault를 직접 부르는 4개 동작(save / ingest / query / lint).
**로직은 vault에만 있다** — 이 스킬은 vault 정본(`$KNOT_VAULT/prompts/`·`scripts/`)을
가리키는 얇은 shim이다. 절차 본문을 여기에 옮겨적지 않는다(정본 단일화 — 절차를 고치면
vault 한 곳만 고친다).

## 0. 게이트 (모든 verb 공통 — 먼저)

`$KNOT_VAULT`가 설정돼 있고 실재 디렉토리인지 확인한다:

```bash
[ -n "$KNOT_VAULT" ] && [ -d "$KNOT_VAULT" ] && echo OK || echo NO_VAULT
```

`NO_VAULT`면 사용자에게 "`$KNOT_VAULT`가 설정돼 있지 않습니다 — vault 경로를 셸 환경변수로
설정한 뒤 다시 시도하세요"라고 안내하고 **작업을 중단**한다(no-op). 경로를 추측하지 말 것.

## 1. verb 분기 (사용자 요청으로 판단)

각 verb는 vault 정본을 **읽고 그대로 따른다**. 벤더(Claude / Codex / agy) 무관 — 동일 vault
prompts를 호출한다(벤더 분기 없음).

- **save** — 사용자가 준 자료(텍스트·파일·URL 메모)를 `$KNOT_VAULT/inbox/`에 새 파일로 저장.
  파일명은 짧은 슬러그. 저장만 한다 — 컴파일은 ingest가 한다.
- **ingest** (compile) — `$KNOT_VAULT/prompts/ingest.md`를 정독하고 그 규약을 **그대로 실행**한다
  (inbox/지정 소스 → wiki 컴파일). 절차·단계는 그 파일에 있다.
- **query** (read) — `$KNOT_VAULT/prompts/query.md`를 정독하고 그 규약으로 근거기반 질의응답.
  vault에 근거가 없으면 **지어내지 않고** "근거 없음"이라고 답한다.
- **lint** — `$KNOT_VAULT/scripts/lint.py`를 실행(기계 검사)한 뒤 `$KNOT_VAULT/prompts/lint.md`를
  정독해 의미 진찰(모순·stale·고아·정합)을 수행한다.

## 2. 크로스도구 (벤더중립)

설계상 Claude Code / Codex / agy(Antigravity) 셋 다 같은 vault prompts를 호출한다. 호스트가
스킬을 자동 로드하지 않으면(예: agy의 `activate_skill` 동작이 다를 수 있음) 사용자가
"`$KNOT_VAULT/prompts/<verb>.md`를 정독하고 실행하라"고 직접 지시해도 동일 결과다 — vault 정본이
유일 진실원이기 때문. (agy/Gemini 스킬 활성 경로는 배포 검증에서 확정.)

## Do NOT

- vault 절차 본문을 이 스킬에 복붙하지 말 것 — 항상 `$KNOT_VAULT/prompts/`·`scripts/`를 가리킨다.
- `$KNOT_VAULT` 미설정 시 경로를 추측하거나 임의 폴더에 저장하지 말 것 — 중단·안내.
- query에서 vault에 없는 내용을 지어내지 말 것.

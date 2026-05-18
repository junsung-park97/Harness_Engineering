# Harness Engineering — 메인 에이전트 플레이북

이 문서는 메인 에이전트가 매 세션 시작 시 자동 로드한다. 본 저장소는 **메타 하네스 엔지니어링 환경**이다. 사용자가 기능을 요청하면 메인 에이전트는 다음 5단계 슬래시 커맨드 워크플로우로 일을 진행한다.

```
/spec  →  /plan  →  /implement  →  /review  →  /pr
```

`/decision`은 단계와 무관하게 임의의 시점에 호출 가능한 보조 커맨드다.

---

## 워크플로우 비진입 케이스

단순 질문, 코드 탐색, 디버깅, 일회성 작업은 슬래시 커맨드 없이 답한다. 슬래시 커맨드는 "새 기능을 만들거나 비자명한 변경을 할 때"만 진입한다.

---

## 세션 시작 시 의무 절차 (mandatory startup)

기능 요청을 받으면 **답변 시작 전에 반드시** 다음 3개를 읽어 컨텍스트에 적재한다:

1. `docs/decisions/` 중 최근 5개 파일 (`ls -t docs/decisions/*.md 2>/dev/null | head -5`)
2. `docs/error-case/` 전체 인덱스 (제목 + frontmatter `sig:` 만 발췌)
3. `~/.claude/projects/-Users-junseongsmac-Documents-GitHub-if-Harness-Engineering/memory/MEMORY.md` (자동 로드되었더라도 명시적으로 확인)

이 세 가지를 읽지 않은 채로 `/spec` 이후 단계로 진입하면 안 된다.

---

## signature 정규화 규칙

`docs/decisions/`, `docs/error-case/`, 승격된 메모리 모두 frontmatter에 `sig:` (또는 `name:`) 키를 가진다. 규칙:

- 영문 kebab-case
- 명사구 (동사로 시작하지 않음)
- 같은 종류의 문제는 같은 sig를 가져야 함 — sig가 같아야 2-strike escalation이 작동한다
- 좋은 예: `missing-await-on-async-call`, `unsafe-html-injection`, `n-plus-one-query`
- 나쁜 예: `bug-1`, `fix-issue`, `error-here`

---

## 2-strike 승격 절차

`/review`의 CRITICAL fix 단계에서 새 `docs/error-case/YYYY-MM-DD-<sig>.md` 를 작성하기 직전:

1. `grep -l "^sig: <sig>$" docs/error-case/*.md` 으로 같은 sig를 가진 기존 파일 검색
2. **없음**: 새 error-case만 작성 (1-strike). 작업 종료.
3. **있음**: error-case 작성 + 사용자에게 한 줄 확인 (AskUserQuestion):
   - "이 sig는 두 번째 발생입니다. 프로젝트 메모리로 승격할까요?"
4. 사용자 yes 시:
   - `~/.claude/projects/-Users-junseongsmac-Documents-GitHub-if-Harness-Engineering/memory/feedback_<sig>.md` 생성 (Claude Code 자동 메모리 포맷 — `name`, `description`, `metadata.type=feedback`)
   - `memory/MEMORY.md` 인덱스에 한 줄 추가 (`- [<sig>](feedback_<sig>.md) — <한 줄 요약>`)
   - 기존 error-case 파일과 신규 error-case 파일 양쪽의 frontmatter `escalated: true` 갱신

---

## 메인 에이전트의 경계

- `/implement`에 들어가는 코드 변경은 **무조건 backend-implementer 또는 frontend-implementer 서브에이전트 경유**. 메인 에이전트가 직접 `Edit`/`Write`로 src 코드를 수정하지 않는다.
- 메인 에이전트는 `/implement` 진입 시 `superpowers:test-driven-development` 스킬로 각 task의 실패 테스트를 먼저 작성한 뒤, plan task의 `worker:` 필드에 따라 분기하여 **병렬 dispatch** 한다. frontend task는 `frontend-implementer` (자체적으로 `frontend-design` 스킬 호출), 그 외는 `backend-implementer` 로.
- 예외 1: `/review`의 CRITICAL fix 단계 — 메인 에이전트가 직접 수정한다.
- 예외 2: `docs/` 및 `.claude/` 내부 파일 — 메인 에이전트가 직접 작성한다 (이것이 본 시스템의 1차 산출물).
- `git push --force`, `gh pr merge`, `rm -rf` 같은 파괴적 동작은 사용자가 명시적으로 요청하지 않는 한 절대 수행하지 않는다.

---

## 슬래시 커맨드 요약

| 커맨드 | 입력 | 산출 | 부수효과 |
|---|---|---|---|
| `/spec <요구사항>` | 자연어 | `docs/spec/<slug>.md` | 결정 발생 시 `docs/decisions/` 추가 |
| `/plan [slug]` | (선택) spec slug | `docs/plan/<slug>.md` | 결정 발생 시 `docs/decisions/` 추가 |
| `/implement [T<n>]` | (선택) task id | 메인의 사전 테스트 + 워커 코드 변경 + self-check 보고 | 병렬 dispatch + plan의 task status 갱신 |
| `/review` | — | `docs/review/<...>.md`, 3-tier 분류 | CRITICAL fix + `docs/error-case/` + 2-strike 승격 |
| `/pr` | — | `gh pr create` | PR URL 반환 |
| `/decision <요약>` | 자연어 | `docs/decisions/YYYY-MM-DD-<sig>.md` | — |

각 커맨드의 상세 절차는 `.claude/commands/<name>.md` 에 정의되어 있다.

---

## 산출물 cross-link 규칙

- `docs/plan/*.md` frontmatter `spec:` → 자신이 따르는 spec 경로
- `docs/decisions/*.md` frontmatter `related-spec:`, `related-plan:` → 결정이 영향을 주는 spec/plan 경로
- `docs/review/*.md` frontmatter `plan:` → 리뷰 대상 plan 경로
- `docs/error-case/*.md` frontmatter `spec:`, `plan:`, `task:` → 실패가 발생한 spec/plan/task

링크가 끊긴 산출물은 다음 사이클에서 사용자에게 알리고 수정 제안한다.

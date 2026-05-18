---
description: 메인이 사전 테스트를 작성한 뒤 backend-implementer / frontend-implementer 를 worker 필드에 따라 분기하여 병렬 dispatch. 인자가 없으면 의존이 만족된 모든 pending task를 한 번에 dispatch.
---

# /implement

## 입력

`$ARGUMENTS` — 선택.
- 없음 → 의존이 만족된 모든 `pending` task 자동 선택 (병렬 dispatch)
- `T<n>` → 그 task 1개
- `T<n>,T<m>,...` → 명시된 다중 task 병렬 dispatch (충돌 검사 후)

## 절차

### 1. plan 로드 + 사전 컨텍스트 수집

- `docs/plan/*.md` 중 가장 최근(mtime) 파일을 active plan으로 사용
- 다음을 모두 메모리에 적재 (워커 주입용 공통 컨텍스트):
  - 해당 spec 전문 (`docs/spec/<slug>.md`)
  - `docs/error-case/` 인덱스 (모든 sig + 제목)
  - `~/.claude/projects/-Users-junseongsmac-Documents-GitHub-if-Harness-Engineering/memory/MEMORY.md` 인덱스

### 2. dispatch 대상 선정

- 인자에 task id가 있으면 그 task(들)을 후보로 사용
- 없으면 plan의 task 중 frontmatter/본문 `status: pending` 이며 의존 task가 모두 `done` 인 task를 모두 후보로
- 후보가 없으면 "구현할 pending task 없음" 보고 후 종료

### 3. 충돌 검사

- 후보 task들의 "수정 파일" 집합을 비교
- 두 task의 수정 파일이 겹치면 같은 dispatch 사이클에서 병렬 진행 불가
- 그 경우 의존/번호 순서로 1차 sub-그룹만 이번에 dispatch하고, 나머지는 사용자에게 "T<x>, T<y>는 다음 호출에서 진행" 알림

### 4. 각 task별 사전 테스트 작성 (메인 직접 수행)

각 후보 task에 대해:

1. `Skill(skill="superpowers:test-driven-development")` 호출 — 가이드 적재
2. task의 `목표:` + `Verification:` + acceptance criteria를 근거로 **실패 테스트**를 작성
   - frontend task → 컴포넌트/페이지의 동작 또는 렌더 결과를 검증하는 테스트 (Vitest + React Testing Library / Playwright / 등 — 프로젝트 컨벤션에 맞춰)
   - backend task → 단위/통합 테스트 (프로젝트 컨벤션에 맞춰)
3. 테스트 러너를 실행해서 `fail` 상태 확인 (구현 전이므로 fail이 정상). 러너가 없거나 환경 미비 시 그 사실을 task 컨텍스트에 명시하고 skip (워커 self-check가 사실상 검증)
4. 작성된 테스트 파일 경로 + 핵심 명세를 해당 task의 dispatch 컨텍스트에 첨부

### 5. task 상태 갱신 (구현 시작)

후보 task들의 `status: pending` → `status: in_progress` 로 plan 파일 Edit.

### 6. 병렬 dispatch (single message multiple Agent calls)

`superpowers:dispatching-parallel-agents` 패턴을 따른다. 한 메시지 안에서 모든 dispatch를 동시에 호출.

각 task의 `worker` 필드에 따라:

- `worker: frontend-implementer` → `Agent(subagent_type="frontend-implementer", ...)`
- `worker: backend-implementer` → `Agent(subagent_type="backend-implementer", ...)`

각 Agent 프롬프트에 다음을 명시 포함:

```
당신은 <worker-name> 서브에이전트입니다. 한 task만 구현하고 self-check 보고서로 종료하세요.

## 시작 시 의무 절차
1. 아래 주입된 spec / plan task 본문을 읽으세요
2. 아래 주입된 error-case 인덱스와 관련 본문을 검토. 같은 sig의 실수를 회피하세요
3. 자동 로드된 feedback_*.md 룰이 있다면 인용 형태로 self-check에 포함
4. **사전 작성된 테스트 파일이 모두 통과하도록 구현하세요. 테스트 파일은 절대 수정하지 마세요.**
<frontend-implementer 한정 추가>
5. `Skill(skill="frontend-design")` 호출 의무. Design Thinking 4축 (Purpose / Tone / Constraints / Differentiation)을 self-check에 명시.
</frontend-implementer 한정 추가>

## 금지 사항
- 다른 서브에이전트 spawn 금지
- git push, PR 생성 금지
- .claude/settings.json 수정 금지
- 테스트 파일 수정/추가 금지
- 본 task가 명시하지 않은 영역을 미리 건드리지 마세요

## self-check 보고서 형식
(서브에이전트 정의 파일의 self-check 양식을 따르세요)

---

## 주입된 컨텍스트

### Spec
<spec 전문>

### Task
<task 본문>

### 의존 task 결과 요약
<있는 경우>

### 사전 작성된 테스트
- 파일: <테스트 파일 경로>
- 명세: <테스트가 검증하는 acceptance 핵심>
- 실행 명령: <예: npm test path/to/test>

### Error-case 인덱스
<sig: 제목 형태의 목록>

### Error-case 관련 본문 (있는 경우)
<발췌>

### MEMORY.md 인덱스
<있는 경우>
```

### 7. 보고서 수신 + 상태 갱신

모든 dispatch가 완료되면 (병렬 Agent 호출이 끝나면):

1. 각 self-check 보고서를 사용자에 표시
2. 메인이 한 번 더 테스트 러너를 실행해서 통과를 확인
3. 통과한 task → plan에서 `status: in_progress` → `status: done`
4. acceptance 미달 또는 테스트 실패한 task → `status: pending`으로 되돌리고 사용자에 알림 + 사유 1줄

### 8. 종료 안내

- 통과 task 목록 + 미달 task 목록(있는 경우) 보고
- "다음 단계: `/review` (지금 변경분 리뷰) 또는 `/implement` (남은 pending task 자동 dispatch)"

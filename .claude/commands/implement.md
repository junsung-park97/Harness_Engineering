---
description: implementer 서브에이전트를 spawn하여 plan의 한 task를 구현. 인자가 없으면 첫 pending task 자동 선택.
---

# /implement

## 입력
`$ARGUMENTS` — 선택. `T1`, `T2`, ... 형식의 task id.

## 절차

### 1. plan 로드
- `docs/plan/*.md` 중 가장 최근(mtime) 파일을 active plan으로 사용
- 인자에 task id가 명시되어 있으면 그 task
- 없으면 plan의 task 중 frontmatter/본문의 `status: pending` 인 첫 번째 task (의존 순서 반영)
- 후보가 없으면 "구현할 pending task 없음" 보고 후 종료

### 2. 사전 컨텍스트 수집 (서브에이전트 주입용)

다음을 모두 메모리에 적재:

- 해당 spec 전문 (`docs/spec/<slug>.md`)
- 해당 task 본문 (plan 파일에서 발췌)
- 이미 완료된 의존 task의 결과 요약 (plan에서 `status: done` 인 의존 task의 결과 노트)
- **`docs/error-case/` 인덱스** (모든 sig + 제목)
- `docs/error-case/` 중 본 task와 키워드가 겹치는 sig의 본문 (관련성 있는 것만 발췌, 무관하면 인덱스만)
- `~/.claude/projects/-Users-junseongsmac-Documents-GitHub-if-Harness-Engineering/memory/MEMORY.md` 인덱스

### 3. task 상태 갱신 (구현 시작)

plan 파일을 Edit 하여 해당 task의 `status: pending` → `status: in_progress`.

### 4. implementer 서브에이전트 spawn

`Agent` 도구로 `subagent_type=general-purpose` 사용 (또는 프로젝트 정의 `implementer` 가 있으면 그것을 사용).

프롬프트에 명시적으로 다음을 포함:

```
당신은 implementer 서브에이전트입니다. 한 task만 구현하고 self-check 보고서로 종료하세요.

## 시작 시 의무 절차
1. 아래 주입된 spec / plan task 본문을 읽으세요
2. 아래 주입된 error-case 인덱스와 관련 본문을 검토하세요. 이 task의 구현에서 같은 sig의 실수를 반복하지 않도록 미리 회피하세요
3. 자동 로드된 feedback_*.md 룰이 있다면 인용 형태로 self-check에 포함하세요

## 금지 사항
- 다른 서브에이전트 spawn 하지 마세요
- git push, PR 생성 하지 마세요
- .claude/settings.json 수정 하지 마세요
- 본 task가 명시하지 않은 영역을 미리 건드리지 마세요

## self-check 보고서 형식 (반드시 끝에 포함)

### 변경된 파일
- path/to/file (어떤 변경)

### Acceptance criteria 검증
- [x] criteria 1: <어떻게 검증했는지 — 명령어/테스트 케이스/수동 확인>
- [x] criteria 2: ...

### 의도적으로 손대지 않은 영역
- <영역> — <이유>

### 적용한 error-case / feedback 룰
- sig:<sig> — <어떻게 회피했는가>
- (없으면 "관련 룰 없음")

---

## 주입된 컨텍스트

### Spec
<spec 전문>

### Task
<task 본문>

### 의존 task 결과 요약
<있는 경우>

### Error-case 인덱스
<sig: 제목 형태의 목록>

### Error-case 관련 본문 (있는 경우)
<발췌>

### MEMORY.md 인덱스
<있는 경우>
```

### 5. 보고서 수신 및 task 상태 갱신

implementer가 self-check 보고서로 종료하면:

- 보고서를 사용자에게 표시
- plan 파일을 Edit 하여 task `status: in_progress` → `status: done`
- 보고서가 acceptance criteria 미달을 시인하면 status를 다시 `pending`으로 되돌리고 사용자에게 알림

### 6. 종료
- "다음 단계: `/review` (지금 변경분 리뷰) 또는 `/implement T<next>` (다음 task)" 안내

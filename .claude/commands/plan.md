---
description: 최신 활성 spec을 읽어 docs/plan/<slug>.md 를 작성. task 단위는 사용자가 명시한 경우 그 단위, 미지정 시 메인 에이전트 임의.
---

# /plan

## 입력
`$ARGUMENTS` — 선택. spec slug 또는 `--tasks "T1: 인증, T2: API"` 형태의 task 단위 명시.

## 절차

### 1. spec 로드
- 인자에 slug가 명시되어 있으면 해당 spec
- 없으면 `docs/spec/*.md` 중 `status: active` 인 최신(mtime) spec 1개
- 후보가 없으면 사용자에게 "먼저 `/spec` 부터 호출하세요" 안내하고 종료

### 2. 사전 컨텍스트 로드
`/spec` 과 동일한 3개 (`docs/decisions/`, `docs/error-case/`, `memory/MEMORY.md`).

### 3. task 분할
**사용자 명시가 있는 경우**: 명시된 단위를 그대로 T1, T2, ... 로 사용.

**미지정 시 메인 에이전트가 결정**. 분할 원칙:

- 한 task = verification 가능한 최소 묶음
- 가능하면 1 task = 1 commit / 1 PR-able 단위
- task 간 의존 관계가 있으면 frontmatter `의존 task:` 에 명시
- task 5개 초과 시 사용자에게 "더 큰 단위로 묶을지" 한 번 확인

### 4. plan 작성
`docs/plan/<slug>.md` (spec과 동일 slug) 를 다음 포맷으로 작성:

```markdown
---
slug: <spec와 동일>
created: YYYY-MM-DD
spec: ../spec/<slug>.md
---

# <기능명> 일정

## 개요
<spec 요약 한 단락>

## Tasks

### T1. <task 제목>
- 목표:
- 수정 파일:
- 의존 task: (없으면 — )
- Verification: <어떻게 검증할지 — 실행 명령, 테스트 케이스, 수동 확인 절차>
- status: pending

### T2. ...

## 실행 순서
T1 → T2 → ... (의존 그래프 반영)
```

### 5. 의사결정 적재
계획 단계에서 발생한 결정도 `/spec` 과 동일하게 `docs/decisions/YYYY-MM-DD-<sig>.md` 에 적재. frontmatter의 `related-plan:` 필드를 채운다.

### 6. 종료
- 작성한 파일 경로와 task 목록 보고
- "다음 단계: `/implement` 또는 `/implement T1`" 안내

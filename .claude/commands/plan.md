---
description: plan 스킬(writing-plans 디시플린)을 호출하여 최신 활성 spec을 구현 계획으로 변환. 하네스 규격에 맞춰 docs/plan/<slug>.md 에 task 단위로 저장. 각 task에 worker 필드 할당.
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

`/spec` 과 동일한 3개 (`docs/decisions/`, `docs/error-case/`, `memory/MEMORY.md`) 를 읽어 plan 스킬 호출 시 입력으로 사용.

### 3. plan 스킬 호출

`Skill(skill="plan")` 을 호출한다. 스킬 본문의 절차(파일 구조 매핑 → bite-sized task 분할 → 각 step의 실제 코드/명령 명시 → self-review → 핸드오프)를 그대로 따른다.

**단, 다음 user preference 를 스킬에 명시적으로 override 로 지시한다** (스킬 본문 "User preferences for plan location override this default" 절을 활용):

#### 저장 경로

- 스킬 디폴트: `docs/plans/YYYY-MM-DD-<feature-name>.md`
- **하네스 override**: `docs/plan/<slug>.md` (단수 `plan/`, spec과 동일 slug)

#### frontmatter 및 본문 골격

스킬 디폴트의 헤더(`# [Feature Name] Implementation Plan` + Goal/Architecture/Tech Stack)를 하네스 표준으로 교체:

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
- worker: backend-implementer | frontend-implementer
- Verification: <어떻게 검증할지 — 실행 명령, 테스트 케이스, 수동 확인 절차>
- status: pending

### T2. ...

## 실행 순서
T1 → T2 → ... (의존 그래프 반영)
```

#### task 분할 단위 (스킬 본문의 "Bite-Sized Task Granularity" 의 변형)

- 스킬 디폴트: 한 task가 5개 step(테스트 작성 → fail 확인 → 구현 → pass 확인 → commit)으로 구성
- **하네스 override**: 한 task = verification 가능한 최소 묶음 = 1 commit / 1 PR-able 단위. step 분할은 task 본문 안에 풀어쓰지 않음 (실패 테스트는 `/implement` 단계에서 메인이 사전 작성하고, 구현은 워커가 수행하므로 step-level은 워커의 self-check가 담당)
- 사용자 명시가 있으면 그 단위 그대로, 없으면 메인이 임의 분할
- task 5개 초과 시 사용자에게 "더 큰 단위로 묶을지" 한 번 확인

#### worker 할당 규칙 (각 task에 필수)

- `frontend-implementer` — UI/컴포넌트/페이지/스타일/HTML·CSS/클라이언트 인터랙션 (React, Vue, HTML, CSS, Tailwind 등 시각 산출물)
- `backend-implementer` — 그 외 전부 (서버, API, DB, 스크립트, 빌드, 설정, `docs/`, `.claude/` 등)
- 한 task에 frontend와 backend가 섞이면 둘로 분할
- 모호한 경우 사용자에게 한 줄 확인 ("T<n>은 UI 작업입니까?")

#### 핸드오프

- 스킬 디폴트: "subagent-dispatching 스킬로 핸드오프 (실행)"
- **하네스 override**: 핸드오프 메시지는 "다음 단계: `/implement` 또는 `/implement T1`" 으로 안내. 사용자가 `/implement` 슬래시 커맨드를 호출해야 메인이 사전 테스트 작성 + worker 분기 병렬 dispatch 사이클을 시작한다.

### 4. 의사결정 적재 (스킬 종료 후 같은 턴에 처리)

계획 단계에서 발생한 결정도 `/spec` 과 동일하게 `docs/decisions/YYYY-MM-DD-<sig>.md` 에 적재. frontmatter의 `related-plan:` 필드를 채운다.

### 5. 종료

- 작성한 파일 경로와 task 목록 (T<n> + worker 필드 포함) 보고
- "다음 단계: `/implement` 또는 `/implement T1`" 안내

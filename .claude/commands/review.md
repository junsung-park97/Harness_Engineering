---
description: code-review 스킬로 1차 리뷰 후 결과를 CRITICAL/FUTURE_RISK/IMPROVEMENT 3-tier로 재분류. CRITICAL은 즉시 fix하고 docs/error-case/ 적재 + 2-strike 시 메모리 승격.
---

# /review

## 입력
`$ARGUMENTS` — 사용 안 함.

## 절차

### 1. 변경분 확인
- `git diff` (커밋 안 된 것 포함) + `git diff <main>...HEAD` 로 변경 파일 식별
- 변경이 없으면 "리뷰할 변경 없음" 보고 후 종료

### 2. code-review 스킬 호출

```
Skill(skill="code-review:code-review", args="<현재 변경분에 대해 코드 리뷰>")
```

스킬이 반환한 raw findings를 받는다.

### 3. 컨텍스트 결합

다음을 합쳐 메인 에이전트가 직접 판단:

- code-review 스킬의 raw findings
- `git diff` 본문
- 활성 spec (`docs/spec/<slug>.md`)
- 활성 plan (`docs/plan/<slug>.md`)
- `docs/decisions/` 중 본 작업 관련 decision

### 4. 3-tier 재분류 규칙

각 finding을 다음 기준으로 분류:

- **CRITICAL** — 지금 동작 위험. 즉시 fix 대상.
  - 빌드 실패 / 타입 에러 / import 깨짐
  - 실행 시 크래시 가능 (null deref, undefined ref 등)
  - 보안 결함 (XSS, SQLi, 시크릿 노출 등)
  - spec의 acceptance criteria 미달
  - 데이터 손실 가능성
- **FUTURE_RISK** — 추후 위험 제기. 기록만.
  - 예외 처리 누락 (실패 경로 미정의)
  - N+1 쿼리 등 확장성 병목
  - 테스트 누락
  - 동시성 가정
- **IMPROVEMENT** — 개선되면 좋음. 기록만.
  - 네이밍
  - 중복 / 추상화 기회
  - 주석/문서 누락

분류 시 confidence가 낮은 항목은 한 단계 낮춰 분류한다 (보수적).

### 5. 리뷰 보고서 작성

`docs/review/YYYY-MM-DD-<slug>-review.md` 작성:

```markdown
---
slug: <spec slug>
date: YYYY-MM-DD
plan: ../plan/<slug>.md
---

# <기능명> 리뷰 — YYYY-MM-DD

## CRITICAL (즉시 fix)
### sig: <kebab>
- 근거: <왜 critical인지 — 빌드 깨짐, spec 미달, 등>
- 위치: <파일:라인>
- fix: <메인 에이전트가 적용한 fix 요약>
- error-case-log: ../error-case/YYYY-MM-DD-<sig>.md

## FUTURE_RISK
### sig: <kebab>
- 근거:
- 위치:

## IMPROVEMENT
### sig: <kebab>
- 근거:
- 위치:
```

### 6. CRITICAL fix 수행

CRITICAL 항목 각각에 대해:

1. 메인 에이전트가 직접 Edit/Write 로 코드 수정
2. fix 직후 다음 절차로 error-case 적재 + 2-strike 체크

### 7. error-case 작성 + 2-strike 승격

CRITICAL fix 후 각 sig에 대해:

#### 7-1. 동일 sig 검색
```bash
grep -l "^sig: <sig>$" docs/error-case/*.md
```

#### 7-2. 새 error-case 작성

`docs/error-case/YYYY-MM-DD-<sig>.md` 를 다음 포맷으로 작성:

```markdown
---
sig: <kebab>
date: YYYY-MM-DD
spec: ../spec/<slug>.md
plan: ../plan/<slug>.md
task: T<n>
escalated: false
---

# YYYY-MM-DD — <에러 한 줄 요약>

## Context
<어떤 task에서, 어떤 파일에서, 어떤 입력/조건 하에 발생했나>

## 상세 원인
<코드 레벨에서 무엇이 잘못되었는가, 왜 발생했는가>

## Fix
<메인 에이전트가 어떻게 고쳤는가 — diff 핵심만>

## 다음 implementer가 다르게 해야 하는 것 (한 줄 룰)
> <kebab-rule-statement>
```

#### 7-3. 2-strike 승격

7-1의 grep 결과에 기존 파일이 **있었으면** (= 이번이 두 번째 발생):

1. 사용자에게 AskUserQuestion:
   - "sig `<sig>`는 두 번째 발생입니다. 프로젝트 메모리(`~/.claude/projects/.../memory/feedback_<sig>.md`)로 승격할까요?"
   - 옵션: "승격 (Recommended)" / "이번만 기록"
2. 승격 선택 시:
   - `~/.claude/projects/-Users-junseongsmac-Documents-GitHub-if-Harness-Engineering/memory/feedback_<sig>.md` 작성 (Claude Code 자동 메모리 포맷):
     ```markdown
     ---
     name: <sig>
     description: <한 줄 요약>
     metadata:
       type: feedback
     ---

     <룰 문장>

     **Why:** 본 sig는 docs/error-case/ 에서 두 번 발생함. 관련 파일:
     - <기존 error-case 경로>
     - <신규 error-case 경로>

     **How to apply:** <어떤 task/파일/상황에서 이 룰이 적용되는가>
     ```
   - `~/.claude/projects/.../memory/MEMORY.md` 에 한 줄 추가:
     ```
     - [<sig>](feedback_<sig>.md) — <한 줄 요약>
     ```
   - 양쪽 error-case 파일의 frontmatter `escalated: false` → `escalated: true`

### 8. 종료
- CRITICAL 몇 건 fix, FUTURE_RISK 몇 건 기록, IMPROVEMENT 몇 건 기록, 승격 여부 요약
- "다음 단계: `/pr` (PR 생성)" 안내

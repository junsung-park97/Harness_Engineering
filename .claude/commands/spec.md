---
description: spec 스킬(브레인스토밍 디시플린)을 호출하여 사용자 요구사항을 spec으로 변환. 하네스 규격에 맞춰 docs/spec/<slug>.md 에 저장하고, 비자명한 의사결정은 같은 턴에 docs/decisions/ 에도 적재.
---

# /spec

## 입력
`$ARGUMENTS` — 자연어 요구사항. 비어 있으면 사용자에게 무엇을 명세할지 한 번 묻고 종료한다.

## 절차

### 1. 사전 컨텍스트 로드 (생략 금지)

스킬을 호출하기 전에 다음을 먼저 읽어 컨텍스트에 적재한다 — 스킬의 "Explore project context" 단계의 입력이 된다:

- `docs/decisions/` 중 최근 5개 (`ls -t docs/decisions/*.md 2>/dev/null | head -5` 후 각각 Read)
- `docs/error-case/` 의 모든 `.md` (제목 + frontmatter만 발췌)
- `~/.claude/projects/-Users-junseongsmac-Documents-GitHub-if-Harness-Engineering/memory/MEMORY.md` (있다면)

### 2. spec 스킬 호출

`Skill(skill="spec")` 을 호출한다. 스킬 본문의 절차(클라리파잉 질문 → 2-3 approaches → 디자인 섹션별 승인 → spec 작성 → self-review → 사용자 리뷰 → writing-plans 핸드오프)를 그대로 따른다.

**단, 다음 user preference 를 스킬에 명시적으로 override 로 지시한다** (스킬 본문 "User preferences for spec location override this default" 절을 활용):

#### 저장 경로

- 스킬 디폴트: `docs/specs/YYYY-MM-DD-<topic>-design.md`
- **하네스 override**: `docs/spec/<slug>.md` (단수 `spec/`, slug는 영문 kebab-case 명사구)
- `docs/spec/<slug>.md` 가 이미 존재하면 사용자에게 (신규 slug / 기존 갱신 / 폐기 후 신규) 중 선택을 받는다

#### frontmatter 및 본문 골격

스킬 디폴트의 자유 형식 대신 하네스 표준 골격을 사용한다:

```markdown
---
slug: <kebab-slug>
created: YYYY-MM-DD
status: active
---

# <기능명>

## Context
<왜 이 기능이 필요한가. 사용자 요청 원문을 인용으로 포함>

## Goals
- ...

## Non-goals
- ...

## Acceptance Criteria
- [ ] 테스트 가능한 조건 1
- [ ] 테스트 가능한 조건 2

## API / Data Model 스케치
<공개 인터페이스, 데이터 스키마, 의존성>

## Open Questions
<현 시점 모르는 것>
```

#### 핸드오프

- 스킬 디폴트: "writing-plans 스킬로 전환"
- **하네스 override**: 핸드오프 메시지는 "다음 단계: `/plan`" 으로 안내. 사용자가 `/plan` 슬래시 커맨드를 호출해야 plan 스킬이 트리거된다.

### 3. 의사결정 적재 (스킬 종료 후 같은 턴에 처리)

spec 작성 도중 다음 중 하나라도 발생하면 같은 턴에 `docs/decisions/YYYY-MM-DD-<sig>.md` 를 추가 작성한다:

- 두 개 이상의 선택지 중 하나를 고름
- spec에 명시되지 않은 가정을 도입함 (예: "결제 게이트웨이는 Stripe로 가정")
- 사용자가 명시한 제약을 명문화함 (예: "MVP는 단건 주문만")

decision 포맷:

```markdown
---
sig: <kebab-signature>
date: YYYY-MM-DD
related-spec: ../spec/<slug>.md
---

# YYYY-MM-DD — <결정 한 줄 요약>

## Context
<왜 이 결정이 필요했나>

## 결정
<무엇을 결정했나>

## 영향 받는 파일

## Related
- [[<related-decision-sig>]]
```

### 4. 종료

- 작성한 파일 경로 목록 사용자에게 보고
- "다음 단계: `/plan`" 안내 한 줄

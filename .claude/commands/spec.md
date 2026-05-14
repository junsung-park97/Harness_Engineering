---
description: 사용자의 기능 요구사항을 받아 docs/spec/<slug>.md 작성. 비자명한 의사결정은 같은 턴에 docs/decisions/ 에도 적재.
---

# /spec

## 입력
`$ARGUMENTS` — 자연어 요구사항. 비어 있으면 사용자에게 무엇을 명세할지 한 번 묻고 종료한다.

## 절차

### 1. 사전 컨텍스트 로드 (생략 금지)
다음을 순서대로 읽는다:

- `docs/decisions/` 중 최근 5개 (`ls -t docs/decisions/*.md 2>/dev/null | head -5` 후 각각 Read)
- `docs/error-case/` 의 모든 `.md` (제목 + frontmatter만 발췌)
- `~/.claude/projects/-Users-junseongsmac-Documents-GitHub-if-Harness-Engineering/memory/MEMORY.md` (있다면)

### 2. slug 결정
요구사항을 영문 kebab-case 명사구로 압축. 예: "결제 모듈 카드 결제 추가" → `card-payment`.

`docs/spec/<slug>.md` 가 이미 존재하면 AskUserQuestion 으로 (신규 slug / 기존 갱신 / 폐기 후 신규) 중 선택받는다.

### 3. spec 작성
`docs/spec/<slug>.md` 를 다음 포맷으로 작성:

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

### 4. 의사결정 적재
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

### 5. 종료
- 작성한 파일 경로 목록 사용자에게 보고
- "다음 단계: `/plan`" 안내 한 줄

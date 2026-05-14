---
description: 임의 시점에 의사결정을 docs/decisions/ 에 즉시 기록. 단계와 무관하게 호출 가능.
---

# /decision

## 입력
`$ARGUMENTS` — 한 줄 요약. 예: `/decision MOCK_PG는 env 토글로 단순화한다`.

## 절차

### 1. sig 결정
요약을 영문 kebab-case 명사구로 변환. 예: "MOCK_PG는 env 토글로 단순화" → `mock-pg-env-toggle`.

### 2. 관련 spec/plan 식별
현재 사이클의 활성 spec/plan이 있으면 해당 경로를 frontmatter `related-spec:` / `related-plan:` 에 넣는다. 없으면 비워둔다.

### 3. 파일 작성
`docs/decisions/YYYY-MM-DD-<sig>.md` 를 다음 포맷으로 작성:

```markdown
---
sig: <kebab>
date: YYYY-MM-DD
related-spec: ../spec/<slug>.md   # 또는 빈 값
related-plan: ../plan/<slug>.md   # 또는 빈 값
---

# YYYY-MM-DD — <한 줄 요약>

## Context
<왜 이 결정이 필요했는가 — 대화의 흐름 요약>

## 결정
<무엇을 결정했는가>

## 영향 받는 파일
- <path/to/file> — <어떤 영향>

## Related
- [[<related-decision-sig>]] (없으면 비움)
```

### 4. 사용자에게 보고
- 작성한 파일 경로
- frontmatter cross-link가 채워졌는지 한 줄로 확인

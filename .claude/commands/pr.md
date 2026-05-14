---
description: 현재 브랜치의 spec/plan/review/decisions/error-cases 를 종합해 PR body를 자동 생성하고 gh pr create로 PR을 연다.
---

# /pr

## 입력
`$ARGUMENTS` — 사용 안 함.

## 절차

### 1. 사전 조건 검사

다음 중 하나라도 위반이면 사용자에게 알리고 중단:

- `git status` 가 clean하지 않음 (커밋 안 된 변경 있음) → "먼저 commit 하세요" 안내
- 현재 브랜치가 `main` 또는 `master` → "별도 feature 브랜치에서 호출하세요" 안내
- `git log <main>..HEAD` 이 비어 있음 (커밋 없음) → "커밋 후 호출하세요"

### 2. 산출물 수집

- 활성 spec: `docs/spec/*.md` 중 `status: active` 인 가장 최근 1개
- 활성 plan: 동일 slug의 `docs/plan/*.md`
- 최신 review: `docs/review/<slug>-*.md` 중 가장 최근 1개 (있으면)
- 이번 사이클의 decisions: 현재 브랜치 커밋에서 추가/수정된 `docs/decisions/*.md` (`git diff <main>...HEAD --name-only` 로 식별)
- 이번 사이클의 error-cases: 동일 방식으로 식별

### 3. PR title 결정

- 형식: `<slug>: <spec의 H1 제목>` (70자 미만)
- 70자 초과 시 spec 제목 절단

### 4. PR body 생성

```markdown
## 무엇 (What)

<spec Goals 섹션을 불릿 1-3개로 요약>

### Acceptance Criteria
<spec의 acceptance criteria 체크리스트 그대로 (✓는 review에서 확인된 것)>

## 왜 (Why)

<spec Context 섹션 요약>

### 관련 결정
- [docs/decisions/<file1>](docs/decisions/<file1>) — <한 줄 요약>
- ...

## 어떻게 (How)

<plan의 task 목록을 1줄씩 요약>

### 핵심 결정
<중요한 trade-off나 결정 요약 2-3줄>

## 검증

<plan의 task별 Verification 요약 + review 3-tier 카운트>

- CRITICAL: <N건> (모두 fix됨)
- FUTURE_RISK: <N건> (기록됨, 후속 라운드)
- IMPROVEMENT: <N건> (기록됨)

## 관련 문서

- spec: [docs/spec/<slug>.md](docs/spec/<slug>.md)
- plan: [docs/plan/<slug>.md](docs/plan/<slug>.md)
- review: [docs/review/<...>.md](docs/review/<...>.md)
- error-cases (이번 사이클):
  - [docs/error-case/<...>.md](docs/error-case/<...>.md)
```

### 5. gh pr create 실행

HEREDOC 으로 body 전달:

```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
<위 본문>
EOF
)"
```

### 6. 금지 사항

- `git push --force` 금지 (사용자 명시 요청 없는 한)
- `gh pr merge` 금지
- 자동 머지/auto-merge 플래그 금지

### 7. 종료

- gh가 반환한 PR URL을 사용자에게 보여준다.
- "리뷰 받은 후 직접 머지하세요" 안내.

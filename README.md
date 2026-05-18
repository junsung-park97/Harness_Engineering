# Harness Engineering

A two-tier learning harness for Claude Code: **main agent** plans, writes failing tests, and reviews; **backend-implementer / frontend-implementer subagents** write code in parallel to pass those tests; mistakes caught in review are recorded as feedback that immediately promotes to long-term memory — so the next worker invocation cannot repeat them.

## Workflow

```
User: "Implement feature X"
        │
        ▼
┌──────────────────────────────────────────────────────────────┐
│                       MAIN AGENT                             │
│                                                              │
│  (1) Spec   →   (2) Plan   →   (3) Review report             │
│                                       │                      │
│                                       │ mistake found?       │
│                                       ▼                      │
│                              /log-review-finding             │
│                                       │                      │
└────────┬──────────────────────────────┼──────────────────────┘
         │ test-first + dispatch by worker field         │
         ▼                                                ▼
┌─────────────────────────────────┐         decisions.jsonl  (kind=review_finding,
│  BACKEND-IMPLEMENTER   (||)     │              │              type=feedback)
│  FRONTEND-IMPLEMENTER  (||)     │              │ promote at count=1 (immediate)
│  subagents (parallel dispatch)  │              ▼
│                                 │      memory/feedback_<sig>.md   ◀─── loaded by
│  (a) Load memory                │              │                      every future
│  (b) Skill(frontend-design)     │              │                      worker run
│      [frontend only]            │              ▼
│  (c) Implement to pass tests    │      MEMORY.md index update
│  (d) Self-check + Report        │
└────────┬────────────────────────┘
         │ next invocation reads the new feedback rule
         ▼
   Same mistake cannot recur
```

## Two-tier responsibilities

| Tier | Owner | Tools | Outputs |
|---|---|---|---|
| **Strategy** | Main agent | Plan mode, `/log-decision`, `/log-review-finding`, `/recall-incidents`, `/why-did-we`, `superpowers:test-driven-development` | Specs, plans, **pre-written failing tests**, review verdicts, recorded findings |
| **Execution (backend)** | `backend-implementer` subagent | Read, Edit, Write, Bash, Grep, Glob | Code changes that pass main's tests + self-check |
| **Execution (frontend)** | `frontend-implementer` subagent | Read, Edit, Write, Bash, Grep, Glob, **Skill** (`frontend-design`) | UI code (Design Thinking + Aesthetics) that passes main's tests + self-check |
| **Analysis** | `incident-analyst` subagent | Read, Bash, Grep, Glob | Pattern reports, classification proposals (no writes) |

## Memory layers

| Layer | Storage | Lifetime | Loaded by |
|---|---|---|---|
| Raw facts | `.harness/logs/*.jsonl` | Indefinite, gitignored | `inject-recent` hook (per prompt), skills (on demand) |
| Long-term rules | `~/.claude/projects/.../memory/feedback_*.md` | Permanent | Claude Code auto-memory (session start), backend-implementer + frontend-implementer startup sequence (every invocation) |

## Promotion thresholds

- `/log-decision` type=feedback — **3 occurrences** of the same signature trigger promotion. For ambient corrections that build up over time.
- `/log-review-finding` — **1 occurrence** triggers immediate promotion. For explicit, post-review mistakes that the loop must close on right away.

## File map

```
CLAUDE.md                                    main agent playbook (always loaded)
.claude/
  settings.json                              hook bindings (PostToolUse, Stop, UserPromptSubmit, SubagentStop)
  agents/
    backend-implementer.md                   non-UI implementation subagent (pass main's tests)
    frontend-implementer.md                  UI implementation subagent (invokes frontend-design skill)
    incident-analyst.md                      the analysis subagent
  skills/
    frontend-design/SKILL.md                 UI aesthetics guide loaded by frontend-implementer
    log-decision/SKILL.md                    record a typed, signature-tagged decision (promote at 3x)
    log-review-finding/SKILL.md              record a mistake from review (promote at 1x)
    recall-incidents/SKILL.md                query failure log
    why-did-we/SKILL.md                      query decision log
.harness/
  bin/
    log-bash-result.py                       PostToolUse: capture Bash failures into incidents.jsonl
    log-decision.py                          Stop: capture decision-shaped assistant text into decisions.jsonl
    inject-recent.py                         UserPromptSubmit: render <harness-memory> block from logs
  logs/
    incidents.jsonl                          (gitignored) failures
    decisions.jsonl                          (gitignored) decisions + review findings
```

## Quick start

1. Open this directory in Claude Code.
2. First time it starts, Claude Code will ask permission to load `.claude/settings.json` (project-local hooks). Approve.
3. Ask the main agent to implement something non-trivial. It will follow `CLAUDE.md`: write a spec, plan with `worker:` assignment per task, write failing tests with `superpowers:test-driven-development`, dispatch `backend-implementer` and `frontend-implementer` in parallel, review the reports.
4. If you spot a mistake in a worker's output, tell the main agent: "그 부분 잘못됐어, signature는 <kebab> 로 기록해줘". The main agent calls `/log-review-finding`, which promotes immediately.
5. On the next worker invocation for the same scope, verify its startup contract lists your recorded signature. If it does, the loop is closed.

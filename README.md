# Harness Engineering

A two-tier learning harness for Claude Code: **main agent** plans and reviews, **implementer subagent** writes the code, and mistakes caught in review are recorded as feedback that immediately promotes to long-term memory — so the next implementer invocation cannot repeat them.

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
         │ spawn with spec+plan         │
         ▼                              ▼
┌──────────────────┐         decisions.jsonl  (kind=review_finding,
│  IMPLEMENTER     │              │             type=feedback)
│  subagent        │              │ promote at count=1 (immediate)
│                  │              ▼
│  (a) Load        │     memory/feedback_<sig>.md   ◀─── loaded by
│      memory      │              │                      every future
│  (b) Implement   │              │                      implementer run
│  (c) Self-check  │              ▼
│  (d) Report      │     MEMORY.md index update
└────────┬─────────┘
         │ next invocation reads the new feedback rule
         ▼
   Same mistake cannot recur
```

## Two-tier responsibilities

| Tier | Owner | Tools | Outputs |
|---|---|---|---|
| **Strategy** | Main agent | Plan mode, `/log-decision`, `/log-review-finding`, `/recall-incidents`, `/why-did-we` | Specs, plans, review verdicts, recorded findings |
| **Execution** | `implementer` subagent | Read, Edit, Write, Bash, Grep, Glob | Code changes + structured report with self-check |
| **Analysis** | `incident-analyst` subagent | Read, Bash, Grep, Glob | Pattern reports, classification proposals (no writes) |

## Memory layers

| Layer | Storage | Lifetime | Loaded by |
|---|---|---|---|
| Raw facts | `.harness/logs/*.jsonl` | Indefinite, gitignored | `inject-recent` hook (per prompt), skills (on demand) |
| Long-term rules | `~/.claude/projects/.../memory/feedback_*.md` | Permanent | Claude Code auto-memory (session start), implementer startup sequence (every invocation) |

## Promotion thresholds

- `/log-decision` type=feedback — **3 occurrences** of the same signature trigger promotion. For ambient corrections that build up over time.
- `/log-review-finding` — **1 occurrence** triggers immediate promotion. For explicit, post-review mistakes that the loop must close on right away.

## File map

```
CLAUDE.md                                    main agent playbook (always loaded)
.claude/
  settings.json                              hook bindings (PostToolUse, Stop, UserPromptSubmit, SubagentStop)
  agents/
    implementer.md                           the implementation subagent
    incident-analyst.md                      the analysis subagent
  skills/
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
3. Ask the main agent to implement something non-trivial. It will follow `CLAUDE.md`: write a spec, plan, spawn `implementer`, review the report.
4. If you spot a mistake in the implementer's output, tell the main agent: "그 부분 잘못됐어, signature는 <kebab> 로 기록해줘". The main agent calls `/log-review-finding`, which promotes immediately.
5. On the next implementer invocation for the same scope, verify its startup contract lists your recorded signature. If it does, the loop is closed.

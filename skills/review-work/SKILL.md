---
name: review-work
description: "5-agent parallel post-implementation review"
---

# review-work

**Description:** Post-implementation review orchestrator. Launches 5 parallel background sub-agents: Oracle (goal/constraint verification), Oracle (code quality), Oracle (security), unspecified-high (hands-on QA execution), unspecified-high (context mining from GitHub/git/Slack/Notion). All must pass for review to pass. MUST USE after completing any significant implementation work. Triggers: 'review work', 'review my work', 'review changes', 'QA my work', 'verify implementation'.

---

# Review Work - 5-Agent Parallel Review Orchestrator

Launch 5 specialized sub-agents in parallel to review completed implementation work from every angle. All 5 must pass for the review to pass.

| # | Agent | Type | Role | Focus |
|---|-------|------|------|-------|
| 1 | Goal Verifier | Oracle | Did we build what was asked? | MAIN |
| 2 | QA Executor | unspecified-high | Does it actually work? | MAIN |
| 3 | Code Reviewer | Oracle | Is the code well-written? | MAIN |
| 4 | Security Auditor | Oracle | Is it secure? | SUB |
| 5 | Context Miner | unspecified-high | Did we miss any context? | MAIN |

---

## Phase 0: Gather Review Context

Collect: GOAL, CONSTRAINTS, BACKGROUND, CHANGED_FILES, DIFF, FILE_CONTENTS, RUN_COMMAND

```bash
git diff --name-only HEAD~1
git diff HEAD~1
```

**NEVER CHECKOUT A PR BRANCH IN THE MAIN WORKTREE. USE `git worktree add`.**

---

## Phase 1: Launch All 5 Agents

Launch ALL 5 in a single turn. Every agent uses `run_in_background=true`.

### Agent 1: Goal & Constraint Verification (Oracle)

Reviews: Goal completeness, constraint compliance, requirement gaps, over-engineering, edge cases, behavioral correctness.

Output:
```
<verdict>PASS or FAIL</verdict>
<goal_breakdown>Each sub-requirement with ACHIEVED/MISSED/PARTIAL</goal_breakdown>
<constraint_compliance>Each constraint with evidence</constraint_compliance>
<blocking_issues>MUST fix items</blocking_issues>
```

### Agent 2: QA via App Execution (unspecified-high)

Process: Brainstorm scenarios (15-30+) → Augment (add 5+) → Create task list (P0/P1/P2) → Execute systematically → Compile results.

For web apps use playwright/dev-browser. For CLI tools run commands. For APIs use curl/httpie.

Output:
```
<verdict>PASS or FAIL</verdict>
<scenario_coverage>Total: N, P0: X passed/Y tested, P1: ..., P2: ...</scenario_coverage>
<test_results>Each test with PASS/FAIL, steps, expected, actual, evidence</test_results>
```

### Agent 3: Code Quality Review (Oracle)

Reviews: Correctness, pattern consistency, naming/readability, error handling, type safety, performance, abstraction level, testing, API design, tech debt.

Severity: CRITICAL (bugs/crashes), MAJOR (should fix before merge), MINOR (worth improving), NITPICK (style).

Output:
```
<verdict>PASS or FAIL</verdict>
<findings>Each with severity, category, file, evidence, suggestion</findings>
<blocking_issues>CRITICAL + MAJOR items</blocking_issues>
```

### Agent 4: Security Review (Oracle) - SUB

Checklist: Input validation, auth/authz, secrets/credentials, data exposure, dependencies, cryptography, file/path traversal, network security, error leakage, supply chain.

Output:
```
<verdict>PASS or FAIL</verdict>
<severity>CRITICAL/HIGH/MEDIUM/LOW/NONE</severity>
<findings>Each with risk description and remediation</findings>
```

### Agent 5: Context Mining (unspecified-high)

Sources: Git history (log, blame, grep), GitHub issues/PRs, Slack/Notion/Discord (if MCP available), codebase cross-references.

Output:
```
<verdict>PASS or FAIL</verdict>
<discovered_context>Each with source, finding, relevance, impact (BLOCKING/IMPORTANT/FYI)</discovered_context>
<missed_requirements>Requirements implementation should address</missed_requirements>
```

---

## Phase 2: Wait & Collect

After launching all 5 agents, end your response. Track completion:

| Agent | Verdict | Notes |
|-------|---------|-------|
| 1. Goal Verification | pending | - |
| 2. QA Execution | pending | - |
| 3. Code Quality | pending | - |
| 4. Security | pending | - |
| 5. Context Mining | pending | - |

Do NOT deliver final report until ALL 5 have completed.

---

## Phase 3: Deliver Verdict

- **PASS**: ALL 5 MAIN agents pass, no blocking issues
- **FAIL with issues**: Any MAIN agent fails → fix issues → re-run review
- **FAIL critical**: CRITICAL security issue → block deployment


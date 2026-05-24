---
name: github-triage
description: "Read-only GitHub issue/PR triage"
---

# github-triage

**Description:** Read-only GitHub triage for issues AND PRs. 1 item = 1 background task (category: quick). Analyzes all open items and writes evidence-backed reports to /tmp/{datetime}/. Every claim requires a GitHub permalink as proof. NEVER takes any action on GitHub - no comments, no merges, no closes, no labels. Reports only. Triggers: 'triage', 'triage issues', 'triage PRs', 'github triage'.

---

# GitHub Triage - Read-Only Analyzer

Read-only GitHub triage orchestrator. Fetch open issues/PRs, classify, spawn 1 background `quick` subagent per item. Each subagent analyzes and writes a report file. ZERO GitHub mutations.

## Architecture

**1 ISSUE/PR = 1 task_create = 1 quick SUBAGENT (background). NO EXCEPTIONS.**

| Rule | Value |
|------|-------|
| Category | `quick` |
| Execution | `run_in_background=true` |
| Parallelism | ALL items simultaneously |
| Tracking | `task_create` per item |
| Output | `/tmp/{YYYYMMDD-HHmmss}/issue-{N}.md` or `pr-{N}.md` |

## Zero-Action Policy (ABSOLUTE)

Subagents MUST NEVER run ANY command that writes or mutates GitHub state.

**FORBIDDEN**: `gh issue comment`, `gh issue close`, `gh pr merge`, `gh pr review`, `gh api -X POST/PUT/PATCH/DELETE`

**ALLOWED**: `gh issue view`, `gh pr view`, `gh api` (GET only), `git log`, `git show`, `git blame`, Write (to `/tmp/` only)

## Evidence Rule (MANDATORY)

Every factual claim in a report MUST include a GitHub permalink:
`https://github.com/{owner}/{repo}/blob/{commit_sha}/{path}#L{start}-L{end}`

- No permalink = no claim. Mark unverifiable as `[UNVERIFIED]`.
- Use commit SHAs only (never branch names like main/master).

## Process

### Phase 0: Setup
```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
REPORT_DIR="/tmp/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$REPORT_DIR"
COMMIT_SHA=$(git rev-parse HEAD)
```

### Phase 1: Fetch All Open Items
```bash
# Issues (paginated)
gh issue list --repo $REPO --state open --limit 500 --json number,title,labels,author,createdAt

# PRs (paginated)  
gh pr list --repo $REPO --state open --limit 500 --json number,title,labels,author,headRefName,baseRefName,isDraft,createdAt
```

**LARGE REPOS**: Process ALL items (no 50-item limit). If 500 open issues → spawn 500 subagents.

### Phase 2: Classify

| Type | Detection |
|------|-----------|
| `ISSUE_QUESTION` | `[Question]`, `?`, "how to", "why does" |
| `ISSUE_BUG` | `[Bug]`, error messages, stack traces |
| `ISSUE_FEATURE` | `[Feature]`, `[Enhancement]`, `Proposal` |
| `ISSUE_OTHER` | Anything else |
| `PR_BUGFIX` | Title starts with `fix`, branch `fix/`, label `bug` |
| `PR_OTHER` | Everything else |

### Phase 3: Spawn Subagents

For each item:
1. `task_create(subject="Triage: #{number} {title}")`
2. `task(category="quick", run_in_background=true, prompt=SUBAGENT_PROMPT)`
3. Store mapping: item_number → { task_id, background_task_id }

### Subagent Prompt Template

```
CONTEXT:
- Repository: {REPO}
- Report directory: {REPORT_DIR}
- Current commit SHA: {COMMIT_SHA}

PERMALINK FORMAT:
https://github.com/{REPO}/blob/{COMMIT_SHA}/{filepath}#L{start}-L{end}

ABSOLUTE RULES:
- NEVER run any gh command that writes to GitHub
- NEVER run git checkout, git fetch, git pull, git switch, git worktree
- Your ONLY writable output: {REPORT_DIR}/{issue|pr}-{number}.md

TASK:
1. Understand the question/bug/feature
2. Search codebase (Grep, Read) for answers
3. Construct permalink for every finding
4. Write report to {REPORT_DIR}/{issue|pr}-{number}.md
```

### Report Format (ISSUE_QUESTION)
```markdown
# Issue #{number}: {title}
**Type:** Question | **Author:** {author} | **Created:** {createdAt}

## Question
[1-2 sentence summary]

## Findings
- Finding with [permalink](https://github.com/{REPO}/blob/{SHA}/path#L42-L58)

## Recommendation
[Suggested answer or next steps]
```

